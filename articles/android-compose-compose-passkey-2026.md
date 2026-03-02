---
title: "Compose Passkey完全ガイド — パスワードレス認証/CredentialManager/FIDO2/生体認証"
emoji: "🔑"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "passkey"]
published: true
---

## この記事で学べること

**Compose Passkey**（パスワードレス認証、Credential Manager、FIDO2、パスキー作成/認証）を解説します。

---

## Credential Manager設定

```groovy
dependencies {
    implementation("androidx.credentials:credentials:1.3.0")
    implementation("androidx.credentials:credentials-play-services-auth:1.3.0")
    implementation("com.google.android.libraries.identity.googleid:googleid:1.1.1")
}
```

---

## パスキー作成

```kotlin
class PasskeyManager @Inject constructor(
    @ApplicationContext private val context: Context
) {
    private val credentialManager = CredentialManager.create(context)

    suspend fun createPasskey(activity: Activity): Result<String> = runCatching {
        val requestJson = """
        {
            "rp": {"name": "MyApp", "id": "example.com"},
            "user": {"id": "${Base64.encodeToString("user123".toByteArray(), Base64.NO_WRAP)}",
                     "name": "user@example.com", "displayName": "ユーザー"},
            "challenge": "${generateChallenge()}",
            "pubKeyCredParams": [{"type": "public-key", "alg": -7}],
            "authenticatorSelection": {"authenticatorAttachment": "platform",
                                       "residentKey": "required"}
        }
        """.trimIndent()

        val request = CreatePublicKeyCredentialRequest(requestJson)
        val result = credentialManager.createCredential(activity, request) as CreatePublicKeyCredentialResponse
        result.registrationResponseJson
    }

    suspend fun signInWithPasskey(activity: Activity): Result<String> = runCatching {
        val requestJson = """
        {
            "challenge": "${generateChallenge()}",
            "rpId": "example.com",
            "userVerification": "required"
        }
        """.trimIndent()

        val request = GetCredentialRequest(listOf(
            GetPublicKeyCredentialOption(requestJson)
        ))
        val result = credentialManager.getCredential(activity, request)
        (result.credential as PublicKeyCredential).authenticationResponseJson
    }

    private fun generateChallenge(): String =
        Base64.encodeToString(ByteArray(32).also { SecureRandom().nextBytes(it) }, Base64.NO_WRAP)
}
```

---

## Compose UI

```kotlin
@Composable
fun PasskeyLoginScreen(viewModel: AuthViewModel = hiltViewModel()) {
    val state by viewModel.authState.collectAsStateWithLifecycle()
    val activity = LocalContext.current as Activity

    Column(Modifier.fillMaxSize().padding(16.dp),
        verticalArrangement = Arrangement.Center,
        horizontalAlignment = Alignment.CenterHorizontally) {

        Icon(Icons.Default.Key, contentDescription = null,
            modifier = Modifier.size(64.dp), tint = MaterialTheme.colorScheme.primary)
        Spacer(Modifier.height(24.dp))
        Text("パスワードレスログイン", style = MaterialTheme.typography.headlineMedium)
        Spacer(Modifier.height(32.dp))

        Button(
            onClick = { viewModel.signInWithPasskey(activity) },
            modifier = Modifier.fillMaxWidth()
        ) {
            Icon(Icons.Default.Fingerprint, contentDescription = null)
            Spacer(Modifier.width(8.dp))
            Text("パスキーでログイン")
        }

        Spacer(Modifier.height(16.dp))

        OutlinedButton(
            onClick = { viewModel.createPasskey(activity) },
            modifier = Modifier.fillMaxWidth()
        ) { Text("パスキーを作成") }

        if (state is AuthState.Error) {
            Spacer(Modifier.height(16.dp))
            Text((state as AuthState.Error).message,
                color = MaterialTheme.colorScheme.error)
        }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `CredentialManager` | 認証情報管理 |
| `CreatePublicKeyCredentialRequest` | パスキー作成 |
| `GetPublicKeyCredentialOption` | パスキー認証 |
| FIDO2 | 標準プロトコル |

- Credential ManagerでFIDO2パスキーを管理
- パスワード不要の生体認証ベースのログイン
- デバイス間同期（Googleアカウント連携）
- フィッシング耐性のある安全な認証

---

8種類のAndroidアプリテンプレート（認証対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose CredentialManager](https://zenn.dev/myougatheaxo/articles/android-compose-compose-credential-manager-2026)
- [Compose Biometric](https://zenn.dev/myougatheaxo/articles/android-compose-compose-biometric-2026)
- [Compose FirebaseAuth](https://zenn.dev/myougatheaxo/articles/android-compose-compose-firebase-auth-2026)
