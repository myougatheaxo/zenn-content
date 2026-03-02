---
title: "生体認証（指紋・顔認証）実装ガイド — BiometricPromptの使い方"
emoji: "🔑"
type: "tech"
topics: ["android", "kotlin", "security", "biometric"]
published: true
---

## この記事で学べること

指紋認証・顔認証をアプリに組み込む方法を解説します。**BiometricPrompt API**を使えば、デバイスの生体認証機能に簡単にアクセスできます。

---

## セットアップ

```kotlin
// build.gradle.kts
dependencies {
    implementation("androidx.biometric:biometric:1.2.0-alpha05")
}
```

---

## 基本実装

```kotlin
class BiometricHelper(private val activity: FragmentActivity) {

    fun authenticate(
        onSuccess: () -> Unit,
        onError: (String) -> Unit
    ) {
        val executor = ContextCompat.getMainExecutor(activity)

        val callback = object : BiometricPrompt.AuthenticationCallback() {
            override fun onAuthenticationSucceeded(
                result: BiometricPrompt.AuthenticationResult
            ) {
                onSuccess()
            }

            override fun onAuthenticationError(
                errorCode: Int,
                errString: CharSequence
            ) {
                onError(errString.toString())
            }

            override fun onAuthenticationFailed() {
                // 認証失敗（指紋不一致など）→ 再試行可能
            }
        }

        val prompt = BiometricPrompt(activity, executor, callback)

        val promptInfo = BiometricPrompt.PromptInfo.Builder()
            .setTitle("認証が必要です")
            .setSubtitle("指紋または顔認証でログインしてください")
            .setNegativeButtonText("キャンセル")
            .build()

        prompt.authenticate(promptInfo)
    }
}
```

---

## 生体認証の利用可否チェック

```kotlin
fun canUseBiometric(context: Context): Boolean {
    val manager = BiometricManager.from(context)
    return when (manager.canAuthenticate(
        BiometricManager.Authenticators.BIOMETRIC_STRONG
    )) {
        BiometricManager.BIOMETRIC_SUCCESS -> true
        BiometricManager.BIOMETRIC_ERROR_NO_HARDWARE -> false
        BiometricManager.BIOMETRIC_ERROR_HW_UNAVAILABLE -> false
        BiometricManager.BIOMETRIC_ERROR_NONE_ENROLLED -> false
        else -> false
    }
}
```

| 結果 | 意味 |
|------|------|
| `BIOMETRIC_SUCCESS` | 利用可能 |
| `ERROR_NO_HARDWARE` | ハードウェアなし |
| `ERROR_HW_UNAVAILABLE` | 一時的に利用不可 |
| `ERROR_NONE_ENROLLED` | 生体情報が未登録 |

---

## Composeとの統合

```kotlin
@Composable
fun BiometricScreen() {
    val context = LocalContext.current
    val activity = context as FragmentActivity
    var authResult by remember { mutableStateOf<String?>(null) }

    Column(
        Modifier.fillMaxSize().padding(16.dp),
        horizontalAlignment = Alignment.CenterHorizontally,
        verticalArrangement = Arrangement.Center
    ) {
        if (authResult == null) {
            Button(onClick = {
                BiometricHelper(activity).authenticate(
                    onSuccess = { authResult = "認証成功" },
                    onError = { authResult = "エラー: $it" }
                )
            }) {
                Text("生体認証でログイン")
            }
        } else {
            Text(authResult!!, style = MaterialTheme.typography.headlineMedium)
        }
    }
}
```

---

## デバイスPIN/パスワードのフォールバック

```kotlin
val promptInfo = BiometricPrompt.PromptInfo.Builder()
    .setTitle("認証が必要です")
    .setSubtitle("生体認証またはデバイスのPINで認証")
    .setAllowedAuthenticators(
        BiometricManager.Authenticators.BIOMETRIC_STRONG or
        BiometricManager.Authenticators.DEVICE_CREDENTIAL
    )
    .build()
```

`DEVICE_CREDENTIAL`を追加すると、生体認証に失敗した場合に**PINやパスワードで認証**できます。

---

## CryptoObject（暗号化連携）

```kotlin
// 暗号化キーを生成
val keyGenerator = KeyGenerator.getInstance(
    KeyProperties.KEY_ALGORITHM_AES, "AndroidKeyStore"
)
keyGenerator.init(
    KeyGenParameterSpec.Builder("my_key",
        KeyProperties.PURPOSE_ENCRYPT or KeyProperties.PURPOSE_DECRYPT)
        .setBlockModes(KeyProperties.BLOCK_MODE_CBC)
        .setEncryptionPaddings(KeyProperties.ENCRYPTION_PADDING_PKCS7)
        .setUserAuthenticationRequired(true)
        .build()
)
val key = keyGenerator.generateKey()

// 生体認証と暗号化を連携
val cipher = Cipher.getInstance("AES/CBC/PKCS7Padding")
cipher.init(Cipher.ENCRYPT_MODE, key)
prompt.authenticate(promptInfo, BiometricPrompt.CryptoObject(cipher))
```

---

## まとめ

- `BiometricPrompt`で指紋・顔認証を実装
- `BiometricManager.canAuthenticate()`で利用可否チェック
- `DEVICE_CREDENTIAL`でPINフォールバック
- `CryptoObject`で暗号化との連携
- テストはエミュレータの指紋設定で可能

---

8種類のAndroidアプリテンプレート（セキュリティ機能追加可能な設計）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [パーミッション完全ガイド](https://zenn.dev/myougatheaxo/articles/android-permission-runtime-2026)
- [DataStore移行ガイド](https://zenn.dev/myougatheaxo/articles/android-datastore-migration-2026)
- [アプリ署名完全ガイド](https://zenn.dev/myougatheaxo/articles/android-signing-release-2026)
