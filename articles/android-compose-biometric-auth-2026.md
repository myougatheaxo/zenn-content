---
title: "生体認証(BiometricPrompt) + Compose連携ガイド"
emoji: "🔏"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "security"]
published: true
---

## この記事で学べること

**BiometricPrompt**を使った指紋/顔認証とCompose UIの連携を解説します。

---

## セットアップ

```kotlin
dependencies {
    implementation("androidx.biometric:biometric:1.2.0-alpha05")
}
```

---

## BiometricPromptのラッパー

```kotlin
class BiometricHelper(private val activity: FragmentActivity) {

    fun canAuthenticate(): Boolean {
        val manager = BiometricManager.from(activity)
        return manager.canAuthenticate(
            BiometricManager.Authenticators.BIOMETRIC_STRONG
        ) == BiometricManager.BIOMETRIC_SUCCESS
    }

    fun authenticate(
        title: String = "認証",
        subtitle: String = "生体認証で本人確認",
        onSuccess: () -> Unit,
        onError: (String) -> Unit
    ) {
        val promptInfo = BiometricPrompt.PromptInfo.Builder()
            .setTitle(title)
            .setSubtitle(subtitle)
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
                // 1回の失敗（指紋認識ミス等）
            }
        }

        BiometricPrompt(activity, callback).authenticate(promptInfo)
    }
}
```

---

## Compose画面との連携

```kotlin
@Composable
fun BiometricLoginScreen() {
    val context = LocalContext.current
    val activity = context as FragmentActivity
    val biometricHelper = remember { BiometricHelper(activity) }

    var isAuthenticated by remember { mutableStateOf(false) }
    var errorMessage by remember { mutableStateOf<String?>(null) }

    if (isAuthenticated) {
        Text("認証成功！", style = MaterialTheme.typography.headlineMedium)
    } else {
        Column(
            modifier = Modifier.fillMaxSize(),
            horizontalAlignment = Alignment.CenterHorizontally,
            verticalArrangement = Arrangement.Center
        ) {
            Icon(
                Icons.Default.Fingerprint,
                contentDescription = null,
                modifier = Modifier.size(64.dp),
                tint = MaterialTheme.colorScheme.primary
            )

            Spacer(Modifier.height(24.dp))

            Button(
                onClick = {
                    biometricHelper.authenticate(
                        onSuccess = { isAuthenticated = true },
                        onError = { errorMessage = it }
                    )
                },
                enabled = biometricHelper.canAuthenticate()
            ) {
                Text("生体認証でログイン")
            }

            errorMessage?.let {
                Spacer(Modifier.height(8.dp))
                Text(it, color = MaterialTheme.colorScheme.error)
            }

            if (!biometricHelper.canAuthenticate()) {
                Text(
                    "生体認証を利用できません",
                    color = MaterialTheme.colorScheme.error,
                    modifier = Modifier.padding(top = 8.dp)
                )
            }
        }
    }
}
```

---

## まとめ

- `BiometricPrompt`で指紋/顔認証を実装
- `BiometricManager.canAuthenticate()`で対応チェック
- `BIOMETRIC_STRONG`で強度の高い認証を要求
- `FragmentActivity`が必要（ComponentActivity単独では不可）
- エラーハンドリングで未対応端末をフォロー
- 機密データアクセス前の認証ゲートとして活用

---

8種類のAndroidアプリテンプレート（セキュリティ設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Social Loginガイド](https://zenn.dev/myougatheaxo/articles/android-compose-social-login-2026)
- [DataStore暗号化](https://zenn.dev/myougatheaxo/articles/android-datastore-2026)
- [セキュリティチェックリスト](https://zenn.dev/myougatheaxo/articles/android-security-checklist-2026)
