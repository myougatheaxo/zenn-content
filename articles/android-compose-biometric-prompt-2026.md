---
title: "生体認証完全ガイド — BiometricPrompt/指紋/顔認証/暗号化連携"
emoji: "🔐"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "auth"]
published: true
---

## この記事で学べること

**生体認証**（BiometricPrompt、指紋認証、顔認証、暗号化キー連携）を解説します。

---

## BiometricPrompt

```kotlin
class BiometricHelper @Inject constructor(
    @ApplicationContext private val context: Context
) {
    fun canAuthenticate(): Boolean {
        val manager = BiometricManager.from(context)
        return manager.canAuthenticate(BiometricManager.Authenticators.BIOMETRIC_STRONG) ==
            BiometricManager.BIOMETRIC_SUCCESS
    }

    fun authenticate(
        activity: FragmentActivity,
        onSuccess: () -> Unit,
        onError: (String) -> Unit
    ) {
        val promptInfo = BiometricPrompt.PromptInfo.Builder()
            .setTitle("生体認証")
            .setSubtitle("指紋または顔認証でログイン")
            .setNegativeButtonText("キャンセル")
            .setAllowedAuthenticators(BiometricManager.Authenticators.BIOMETRIC_STRONG)
            .build()

        val callback = object : BiometricPrompt.AuthenticationCallback() {
            override fun onAuthenticationSucceeded(result: BiometricPrompt.AuthenticationResult) {
                onSuccess()
            }
            override fun onAuthenticationError(errorCode: Int, errString: CharSequence) {
                onError(errString.toString())
            }
            override fun onAuthenticationFailed() {
                onError("認証失敗")
            }
        }

        val prompt = BiometricPrompt(activity, ContextCompat.getMainExecutor(context), callback)
        prompt.authenticate(promptInfo)
    }
}
```

---

## 暗号化連携

```kotlin
class BiometricCrypto @Inject constructor() {
    private val keyStore = KeyStore.getInstance("AndroidKeyStore").apply { load(null) }

    fun generateKey(keyName: String) {
        val keyGenerator = KeyGenerator.getInstance(KeyProperties.KEY_ALGORITHM_AES, "AndroidKeyStore")
        keyGenerator.init(
            KeyGenParameterSpec.Builder(keyName, KeyProperties.PURPOSE_ENCRYPT or KeyProperties.PURPOSE_DECRYPT)
                .setBlockModes(KeyProperties.BLOCK_MODE_CBC)
                .setEncryptionPaddings(KeyProperties.ENCRYPTION_PADDING_PKCS7)
                .setUserAuthenticationRequired(true)
                .setInvalidatedByBiometricEnrollment(true)
                .build()
        )
        keyGenerator.generateKey()
    }

    fun getCipher(keyName: String): Cipher {
        val key = keyStore.getKey(keyName, null) as SecretKey
        return Cipher.getInstance("AES/CBC/PKCS7Padding").apply {
            init(Cipher.ENCRYPT_MODE, key)
        }
    }
}
```

---

## Compose UI

```kotlin
@Composable
fun BiometricLoginScreen(viewModel: AuthViewModel = hiltViewModel()) {
    val context = LocalContext.current as FragmentActivity
    val canUseBiometric by viewModel.canUseBiometric.collectAsStateWithLifecycle(false)

    Column(
        Modifier.fillMaxSize().padding(32.dp),
        horizontalAlignment = Alignment.CenterHorizontally,
        verticalArrangement = Arrangement.Center
    ) {
        Icon(Icons.Default.Fingerprint, null, Modifier.size(80.dp), tint = MaterialTheme.colorScheme.primary)
        Spacer(Modifier.height(24.dp))

        if (canUseBiometric) {
            Button(onClick = { viewModel.authenticate(context) }, Modifier.fillMaxWidth()) {
                Text("生体認証でログイン")
            }
        } else {
            Text("生体認証が利用できません", color = Color.Gray)
        }
    }
}
```

---

## まとめ

| 認証方式 | Authenticator |
|---------|---------------|
| 指紋/顔 | `BIOMETRIC_STRONG` |
| PIN/パターン | `DEVICE_CREDENTIAL` |
| 両方 | 組み合わせ可 |

- `BiometricPrompt`でシステム標準の認証UI
- `canAuthenticate`で事前に利用可能か確認
- `AndroidKeyStore`で生体認証連動の暗号化キー
- `setUserAuthenticationRequired`で認証必須の暗号化

---

8種類のAndroidアプリテンプレート（認証機能対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Google Sign-In](https://zenn.dev/myougatheaxo/articles/android-compose-google-signin-2026)
- [Credential Manager](https://zenn.dev/myougatheaxo/articles/android-compose-credential-manager-2026)
- [暗号化](https://zenn.dev/myougatheaxo/articles/android-compose-encrypted-shared-pref-2026)
