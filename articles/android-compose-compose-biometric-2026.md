---
title: "Biometric認証完全ガイド — 指紋/顔認証/BiometricPrompt/Compose統合"
emoji: "🔒"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "security"]
published: true
---

## この記事で学べること

**Biometric認証**（BiometricPrompt、指紋/顔認証、Compose統合、フォールバック）を解説します。

---

## BiometricPrompt基本

```kotlin
@Composable
fun BiometricAuthScreen() {
    val context = LocalContext.current
    val activity = context as FragmentActivity
    var authResult by remember { mutableStateOf<String?>(null) }

    val biometricPrompt = remember {
        BiometricPrompt(activity, object : BiometricPrompt.AuthenticationCallback() {
            override fun onAuthenticationSucceeded(result: BiometricPrompt.AuthenticationResult) {
                authResult = "認証成功"
            }
            override fun onAuthenticationError(errorCode: Int, errString: CharSequence) {
                authResult = "エラー: $errString"
            }
            override fun onAuthenticationFailed() {
                authResult = "認証失敗"
            }
        })
    }

    val promptInfo = remember {
        BiometricPrompt.PromptInfo.Builder()
            .setTitle("生体認証")
            .setSubtitle("指紋または顔認証でログイン")
            .setNegativeButtonText("キャンセル")
            .setAllowedAuthenticators(BiometricManager.Authenticators.BIOMETRIC_STRONG)
            .build()
    }

    Column(Modifier.padding(16.dp), horizontalAlignment = Alignment.CenterHorizontally) {
        Button(onClick = { biometricPrompt.authenticate(promptInfo) }) {
            Icon(Icons.Default.Fingerprint, null)
            Spacer(Modifier.width(8.dp))
            Text("生体認証でログイン")
        }

        authResult?.let {
            Spacer(Modifier.height(16.dp))
            Text(it, color = if (it == "認証成功") Color(0xFF4CAF50) else Color.Red)
        }
    }
}
```

---

## 利用可能チェック

```kotlin
@Composable
fun BiometricAvailabilityCheck() {
    val context = LocalContext.current
    val biometricManager = BiometricManager.from(context)

    val canAuthenticate = remember {
        when (biometricManager.canAuthenticate(BiometricManager.Authenticators.BIOMETRIC_STRONG)) {
            BiometricManager.BIOMETRIC_SUCCESS -> "利用可能"
            BiometricManager.BIOMETRIC_ERROR_NO_HARDWARE -> "ハードウェアなし"
            BiometricManager.BIOMETRIC_ERROR_HW_UNAVAILABLE -> "一時的に利用不可"
            BiometricManager.BIOMETRIC_ERROR_NONE_ENROLLED -> "未登録"
            else -> "不明"
        }
    }

    Card(Modifier.fillMaxWidth().padding(16.dp)) {
        Column(Modifier.padding(16.dp)) {
            Text("生体認証ステータス", style = MaterialTheme.typography.titleMedium)
            Text(canAuthenticate)
        }
    }
}
```

---

## デバイス認証フォールバック

```kotlin
@Composable
fun BiometricWithFallback() {
    val context = LocalContext.current
    val activity = context as FragmentActivity

    val promptInfo = remember {
        BiometricPrompt.PromptInfo.Builder()
            .setTitle("認証が必要です")
            .setSubtitle("続行するには認証してください")
            .setAllowedAuthenticators(
                BiometricManager.Authenticators.BIOMETRIC_STRONG or
                BiometricManager.Authenticators.DEVICE_CREDENTIAL
            )
            .build()
    }

    Button(onClick = {
        val prompt = BiometricPrompt(activity, object : BiometricPrompt.AuthenticationCallback() {
            override fun onAuthenticationSucceeded(result: BiometricPrompt.AuthenticationResult) {
                // 認証成功後の処理
            }
        })
        prompt.authenticate(promptInfo)
    }) {
        Text("認証して続行")
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `BiometricPrompt` | 認証ダイアログ |
| `PromptInfo` | ダイアログ設定 |
| `BiometricManager` | 利用可能チェック |
| `DEVICE_CREDENTIAL` | PIN/パターンフォールバック |

- `BiometricPrompt`で指紋/顔認証ダイアログ表示
- `BiometricManager`で事前に利用可能性チェック
- `DEVICE_CREDENTIAL`でPIN/パターンにフォールバック
- `BIOMETRIC_STRONG`でClass 3生体認証を要求

---

8種類のAndroidアプリテンプレート（セキュリティ対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [パーミッション管理](https://zenn.dev/myougatheaxo/articles/android-compose-compose-permission-handler-2026)
- [DataStore](https://zenn.dev/myougatheaxo/articles/android-compose-datastore-2026)
- [Accompanist Permissions](https://zenn.dev/myougatheaxo/articles/android-compose-compose-accompanist-permissions-2026)
