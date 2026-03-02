---
title: "Compose CredentialManager完全ガイド — パスキー/パスワード管理/Google認証"
emoji: "🔑"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "auth"]
published: true
---

## この記事で学べること

**Compose CredentialManager**（パスキー認証、パスワード保存、Googleサインイン統合）を解説します。

---

## 基本CredentialManager

```kotlin
// build.gradle
// implementation "androidx.credentials:credentials:1.3.0"
// implementation "androidx.credentials:credentials-play-services-auth:1.3.0"

class AuthManager @Inject constructor(
    @ApplicationContext private val context: Context
) {
    private val credentialManager = CredentialManager.create(context)

    suspend fun signIn(activity: Activity): String? {
        val request = GetCredentialRequest(listOf(
            GetPasswordOption(),
            GetPublicKeyCredentialOption(fetchAuthenticationJson())
        ))

        return try {
            val result = credentialManager.getCredential(activity, request)
            handleSignInResult(result)
        } catch (e: GetCredentialCancellationException) {
            null
        } catch (e: NoCredentialException) {
            null
        }
    }

    private fun handleSignInResult(result: GetCredentialResponse): String? {
        return when (val credential = result.credential) {
            is PasswordCredential -> {
                // パスワード認証
                authenticate(credential.id, credential.password)
            }
            is PublicKeyCredential -> {
                // パスキー認証
                verifyPasskey(credential.authenticationResponseJson)
            }
            else -> null
        }
    }
}
```

---

## パスワード保存

```kotlin
suspend fun savePassword(activity: Activity, username: String, password: String) {
    val credentialManager = CredentialManager.create(activity)

    val request = CreatePasswordRequest(username, password)
    try {
        credentialManager.createCredential(activity, request)
    } catch (e: CreateCredentialCancellationException) {
        // ユーザーがキャンセル
    }
}
```

---

## Compose統合

```kotlin
@Composable
fun CredentialLoginScreen(authManager: AuthManager) {
    val context = LocalContext.current
    val activity = context as Activity
    var isLoading by remember { mutableStateOf(false) }
    var error by remember { mutableStateOf<String?>(null) }

    Column(Modifier.fillMaxSize().padding(16.dp), verticalArrangement = Arrangement.Center) {
        Button(
            onClick = {
                isLoading = true
                CoroutineScope(Dispatchers.Main).launch {
                    val result = authManager.signIn(activity)
                    isLoading = false
                    if (result == null) error = "認証失敗"
                }
            },
            modifier = Modifier.fillMaxWidth(),
            enabled = !isLoading
        ) {
            if (isLoading) CircularProgressIndicator(Modifier.size(20.dp), color = Color.White)
            else { Icon(Icons.Default.Key, null); Spacer(Modifier.width(8.dp)); Text("サインイン") }
        }

        error?.let { Text(it, color = MaterialTheme.colorScheme.error, modifier = Modifier.padding(top = 8.dp)) }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `CredentialManager` | 認証管理 |
| `GetPasswordOption` | パスワード取得 |
| `GetPublicKeyCredentialOption` | パスキー取得 |
| `CreatePasswordRequest` | パスワード保存 |

- `CredentialManager`はパスワード/パスキー/Googleの統合API
- パスキーでパスワードレス認証を実現
- `NoCredentialException`で保存済み認証情報なしを検出
- Android 14+でネイティブサポート、それ以前はPlay Services経由

---

8種類のAndroidアプリテンプレート（認証対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Firebase Auth](https://zenn.dev/myougatheaxo/articles/android-compose-compose-firebase-auth-2026)
- [Compose Biometric](https://zenn.dev/myougatheaxo/articles/android-compose-compose-biometric-2026)
- [Retrofit Authenticator](https://zenn.dev/myougatheaxo/articles/android-compose-retrofit-authenticator-2026)
