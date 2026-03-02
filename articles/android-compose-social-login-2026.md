---
title: "ソーシャルログインガイド — Google Sign-In + Credential Manager"
emoji: "🔑"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "auth"]
published: true
---

## この記事で学べること

**Credential Manager**を使ったGoogle Sign-Inの実装を解説します。

---

## セットアップ

```kotlin
dependencies {
    implementation("androidx.credentials:credentials:1.5.0")
    implementation("androidx.credentials:credentials-play-services-auth:1.5.0")
    implementation("com.google.android.libraries.identity.googleid:googleid:1.1.1")
}
```

---

## Credential Manager初期化

```kotlin
class AuthRepository(private val context: Context) {

    private val credentialManager = CredentialManager.create(context)

    suspend fun signInWithGoogle(): GoogleIdTokenCredential? {
        val googleIdOption = GetGoogleIdOption.Builder()
            .setFilterByAuthorizedAccounts(false)
            .setServerClientId("YOUR_WEB_CLIENT_ID.apps.googleusercontent.com")
            .build()

        val request = GetCredentialRequest.Builder()
            .addCredentialOption(googleIdOption)
            .build()

        return try {
            val result = credentialManager.getCredential(context as Activity, request)
            val credential = result.credential
            if (credential is CustomCredential &&
                credential.type == GoogleIdTokenCredential.TYPE_GOOGLE_ID_TOKEN_CREDENTIAL) {
                GoogleIdTokenCredential.createFrom(credential.data)
            } else null
        } catch (e: GetCredentialException) {
            null
        }
    }
}
```

---

## ViewModel

```kotlin
class AuthViewModel(private val authRepository: AuthRepository) : ViewModel() {

    private val _authState = MutableStateFlow<AuthState>(AuthState.SignedOut)
    val authState: StateFlow<AuthState> = _authState.asStateFlow()

    fun signIn() {
        viewModelScope.launch {
            _authState.value = AuthState.Loading
            val credential = authRepository.signInWithGoogle()
            if (credential != null) {
                _authState.value = AuthState.SignedIn(
                    name = credential.displayName ?: "",
                    email = credential.id,
                    photoUrl = credential.profilePictureUri?.toString()
                )
            } else {
                _authState.value = AuthState.Error("ログインに失敗しました")
            }
        }
    }

    fun signOut() {
        _authState.value = AuthState.SignedOut
    }
}

sealed interface AuthState {
    data object SignedOut : AuthState
    data object Loading : AuthState
    data class SignedIn(val name: String, val email: String, val photoUrl: String?) : AuthState
    data class Error(val message: String) : AuthState
}
```

---

## ログイン画面

```kotlin
@Composable
fun LoginScreen(viewModel: AuthViewModel = viewModel()) {
    val authState by viewModel.authState.collectAsStateWithLifecycle()

    Column(
        Modifier.fillMaxSize().padding(32.dp),
        horizontalAlignment = Alignment.CenterHorizontally,
        verticalArrangement = Arrangement.Center
    ) {
        Text("ログイン", style = MaterialTheme.typography.headlineMedium)
        Spacer(Modifier.height(32.dp))

        when (authState) {
            is AuthState.Loading -> CircularProgressIndicator()
            is AuthState.Error -> {
                Text(
                    (authState as AuthState.Error).message,
                    color = MaterialTheme.colorScheme.error
                )
                Spacer(Modifier.height(16.dp))
            }
            else -> {}
        }

        OutlinedButton(
            onClick = { viewModel.signIn() },
            enabled = authState !is AuthState.Loading,
            modifier = Modifier.fillMaxWidth()
        ) {
            Icon(
                painter = painterResource(R.drawable.ic_google),
                contentDescription = null,
                modifier = Modifier.size(20.dp),
                tint = Color.Unspecified
            )
            Spacer(Modifier.width(8.dp))
            Text("Googleでログイン")
        }
    }
}
```

---

## プロフィール画面

```kotlin
@Composable
fun ProfileScreen(authState: AuthState.SignedIn, onSignOut: () -> Unit) {
    Column(
        Modifier.fillMaxSize().padding(16.dp),
        horizontalAlignment = Alignment.CenterHorizontally
    ) {
        authState.photoUrl?.let { url ->
            AsyncImage(
                model = url,
                contentDescription = "プロフィール画像",
                modifier = Modifier.size(80.dp).clip(CircleShape)
            )
        }
        Spacer(Modifier.height(16.dp))
        Text(authState.name, style = MaterialTheme.typography.titleLarge)
        Text(authState.email, style = MaterialTheme.typography.bodyMedium)
        Spacer(Modifier.height(24.dp))
        OutlinedButton(onClick = onSignOut) { Text("ログアウト") }
    }
}
```

---

## まとめ

- `Credential Manager`が最新のGoogle Sign-In推奨方法
- `GetGoogleIdOption`でGoogleアカウント選択
- `GoogleIdTokenCredential`からユーザー情報取得
- ViewModelで`AuthState` sealed interfaceで状態管理
- IDトークンをバックエンドに送信して検証
- プロフィール画像はCoil `AsyncImage`で表示

---

8種類のAndroidアプリテンプレート（認証対応可能）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Firebase Authガイド](https://zenn.dev/myougatheaxo/articles/android-firebase-auth-2026)
- [生体認証ガイド](https://zenn.dev/myougatheaxo/articles/android-biometric-auth-2026)
- [セキュリティチェックリスト](https://zenn.dev/myougatheaxo/articles/android-security-checklist-2026)
