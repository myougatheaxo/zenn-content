---
title: "Google Sign-In完全ガイド — Credential Manager/ワンタップ/Firebase Auth"
emoji: "🔐"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "auth"]
published: true
---

## この記事で学べること

**Google Sign-In**（Credential Manager、ワンタップUI、Firebase Auth連携、サインアウト）を解説します。

---

## Credential Manager方式（推奨）

```kotlin
class GoogleAuthManager @Inject constructor(
    @ApplicationContext private val context: Context
) {
    private val credentialManager = CredentialManager.create(context)

    suspend fun signIn(activity: Activity): GoogleIdTokenCredential? {
        val googleIdOption = GetGoogleIdOption.Builder()
            .setFilterByAuthorizedAccounts(false)
            .setServerClientId(BuildConfig.GOOGLE_WEB_CLIENT_ID)
            .build()

        val request = GetCredentialRequest.Builder()
            .addCredentialOption(googleIdOption)
            .build()

        return try {
            val result = credentialManager.getCredential(activity, request)
            val credential = result.credential

            if (credential is CustomCredential &&
                credential.type == GoogleIdTokenCredential.TYPE_GOOGLE_ID_TOKEN_CREDENTIAL
            ) {
                GoogleIdTokenCredential.createFrom(credential.data)
            } else null
        } catch (e: GetCredentialCancellationException) {
            null  // ユーザーがキャンセル
        } catch (e: NoCredentialException) {
            null  // 認証情報なし
        }
    }

    suspend fun signOut() {
        credentialManager.clearCredentialState(ClearCredentialStateRequest())
    }
}
```

---

## Firebase Auth連携

```kotlin
class AuthRepository @Inject constructor(
    private val googleAuthManager: GoogleAuthManager,
    private val firebaseAuth: FirebaseAuth
) {
    val currentUser: Flow<FirebaseUser?> = callbackFlow {
        val listener = FirebaseAuth.AuthStateListener { auth ->
            trySend(auth.currentUser)
        }
        firebaseAuth.addAuthStateListener(listener)
        awaitClose { firebaseAuth.removeAuthStateListener(listener) }
    }

    suspend fun signInWithGoogle(activity: Activity): Result<FirebaseUser> {
        val googleCredential = googleAuthManager.signIn(activity)
            ?: return Result.failure(Exception("Google Sign-In cancelled"))

        val firebaseCredential = GoogleAuthProvider.getCredential(googleCredential.idToken, null)

        return try {
            val result = firebaseAuth.signInWithCredential(firebaseCredential).await()
            Result.success(result.user!!)
        } catch (e: Exception) {
            Result.failure(e)
        }
    }

    fun signOut() {
        firebaseAuth.signOut()
    }
}
```

---

## Compose UI

```kotlin
@Composable
fun SignInScreen(viewModel: AuthViewModel = hiltViewModel()) {
    val authState by viewModel.authState.collectAsStateWithLifecycle()
    val activity = LocalContext.current as Activity

    Column(
        Modifier.fillMaxSize().padding(32.dp),
        horizontalAlignment = Alignment.CenterHorizontally,
        verticalArrangement = Arrangement.Center
    ) {
        when (authState) {
            is AuthState.SignedOut -> {
                Text("ログインしてください", style = MaterialTheme.typography.headlineSmall)
                Spacer(Modifier.height(32.dp))
                OutlinedButton(
                    onClick = { viewModel.signInWithGoogle(activity) },
                    modifier = Modifier.fillMaxWidth()
                ) {
                    Icon(painterResource(R.drawable.ic_google), null, Modifier.size(20.dp))
                    Spacer(Modifier.width(8.dp))
                    Text("Googleでログイン")
                }
            }
            is AuthState.SignedIn -> {
                val user = (authState as AuthState.SignedIn).user
                Text("${user.displayName} さん", style = MaterialTheme.typography.titleLarge)
                Spacer(Modifier.height(16.dp))
                Button(onClick = { viewModel.signOut() }) { Text("ログアウト") }
            }
            is AuthState.Loading -> CircularProgressIndicator()
        }
    }
}
```

---

## まとめ

| 手法 | 特徴 |
|------|------|
| Credential Manager | 最新API、推奨 |
| ワンタップ | 最小限のUI |
| Firebase Auth | バックエンド認証 |
| Sign Out | `clearCredentialState` |

- Credential ManagerでGoogle Sign-In（最新推奨API）
- Firebase Authと連携でユーザー管理
- `callbackFlow`で認証状態をリアクティブ監視
- `clearCredentialState`でサインアウト

---

8種類のAndroidアプリテンプレート（認証機能対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Credential Manager](https://zenn.dev/myougatheaxo/articles/android-compose-credential-manager-2026)
- [Firebase Auth](https://zenn.dev/myougatheaxo/articles/android-compose-firebase-auth-2026)
- [OAuth2/PKCE](https://zenn.dev/myougatheaxo/articles/android-compose-oauth-pkce-2026)
