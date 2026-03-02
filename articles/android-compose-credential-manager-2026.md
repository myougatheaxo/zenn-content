---
title: "Credential Manager完全ガイド — パスキー/パスワード管理/Compose連携"
emoji: "🔐"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "security"]
published: true
---

## この記事で学べること

**Credential Manager**（パスキー、パスワード保存/取得、Google Sign-In統合、Compose UI連携）を解説します。

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

## CredentialRepository

```kotlin
class CredentialRepository @Inject constructor(
    @ApplicationContext private val context: Context
) {
    private val credentialManager = CredentialManager.create(context)

    suspend fun signInWithSavedCredentials(activity: Activity): AuthResult {
        val request = GetCredentialRequest.Builder()
            .addCredentialOption(GetPasswordOption())
            .addCredentialOption(
                GetPublicKeyCredentialOption(
                    requestJson = createPasskeyRequestJson()
                )
            )
            .addCredentialOption(
                GetGoogleIdOption.Builder()
                    .setServerClientId(BuildConfig.GOOGLE_CLIENT_ID)
                    .setFilterByAuthorizedAccounts(false)
                    .build()
            )
            .build()

        return try {
            val result = credentialManager.getCredential(activity, request)
            handleCredential(result.credential)
        } catch (e: GetCredentialCancellationException) {
            AuthResult.Cancelled
        } catch (e: NoCredentialException) {
            AuthResult.NoCredentials
        } catch (e: Exception) {
            AuthResult.Error(e.message ?: "Unknown error")
        }
    }

    suspend fun savePassword(activity: Activity, email: String, password: String) {
        val credential = CreatePasswordRequest(email, password)
        credentialManager.createCredential(activity, credential)
    }

    private fun handleCredential(credential: Credential): AuthResult {
        return when (credential) {
            is PasswordCredential -> {
                AuthResult.Success(credential.id, credential.password)
            }
            is PublicKeyCredential -> {
                AuthResult.PasskeySuccess(credential.authenticationResponseJson)
            }
            is CustomCredential -> {
                if (credential.type == GoogleIdTokenCredential.TYPE_GOOGLE_ID_TOKEN_CREDENTIAL) {
                    val googleId = GoogleIdTokenCredential.createFrom(credential.data)
                    AuthResult.GoogleSuccess(googleId.idToken)
                } else {
                    AuthResult.Error("Unsupported credential type")
                }
            }
            else -> AuthResult.Error("Unsupported credential")
        }
    }
}

sealed interface AuthResult {
    data class Success(val email: String, val password: String) : AuthResult
    data class PasskeySuccess(val responseJson: String) : AuthResult
    data class GoogleSuccess(val idToken: String) : AuthResult
    data object Cancelled : AuthResult
    data object NoCredentials : AuthResult
    data class Error(val message: String) : AuthResult
}
```

---

## Compose画面

```kotlin
@Composable
fun LoginScreen(viewModel: LoginViewModel = hiltViewModel()) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()
    val activity = LocalContext.current as Activity

    LaunchedEffect(Unit) {
        viewModel.signInWithSavedCredentials(activity)
    }

    Column(
        Modifier.fillMaxSize().padding(16.dp),
        verticalArrangement = Arrangement.Center,
        horizontalAlignment = Alignment.CenterHorizontally
    ) {
        when (uiState) {
            LoginUiState.Loading -> CircularProgressIndicator()
            is LoginUiState.Error -> {
                Text((uiState as LoginUiState.Error).message, color = MaterialTheme.colorScheme.error)
                Spacer(Modifier.height(16.dp))
                Button(onClick = { viewModel.signInWithSavedCredentials(activity) }) {
                    Text("サインイン")
                }
            }
            LoginUiState.NeedInput -> {
                Button(
                    onClick = { viewModel.signInWithSavedCredentials(activity) },
                    modifier = Modifier.fillMaxWidth()
                ) { Text("保存済み認証情報でサインイン") }
            }
            LoginUiState.Success -> {
                Text("ログイン成功！", style = MaterialTheme.typography.headlineMedium)
            }
        }
    }
}
```

---

## まとめ

| 機能 | 実装 |
|------|------|
| パスワード取得 | `GetPasswordOption` |
| パスキー | `GetPublicKeyCredentialOption` |
| Google Sign-In | `GetGoogleIdOption` |
| パスワード保存 | `CreatePasswordRequest` |

- `CredentialManager`で統一的な認証情報管理
- パスキー/パスワード/Google Sign-Inを一括提示
- ユーザーは最適な方法を選択して認証
- Android 14+ではシステム統合で最適UX

---

8種類のAndroidアプリテンプレート（認証機能対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Firebase Auth](https://zenn.dev/myougatheaxo/articles/android-compose-firebase-auth-2026)
- [生体認証](https://zenn.dev/myougatheaxo/articles/android-compose-biometric-auth-2026)
- [セキュリティ](https://zenn.dev/myougatheaxo/articles/android-compose-security-best-practices-2026)
