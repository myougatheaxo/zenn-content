---
title: "アプリ内メッセージ完全ガイド — Firebase In-App Messaging/カスタムバナー/SnackbarHost"
emoji: "💬"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "firebase"]
published: true
---

## この記事で学べること

**アプリ内メッセージ**（Firebase In-App Messaging、カスタムバナー、SnackbarHost、ダイアログ通知）を解説します。

---

## Firebase In-App Messaging

```kotlin
// build.gradle.kts
dependencies {
    implementation(platform(libs.firebase.bom))
    implementation("com.google.firebase:firebase-inappmessaging-display-ktx")
}

// カスタム表示
class CustomInAppMessageDisplay @Inject constructor() : FirebaseInAppMessagingDisplay {
    private val _message = MutableStateFlow<InAppMessage?>(null)
    val message: StateFlow<InAppMessage?> = _message.asStateFlow()

    override fun displayMessage(
        inAppMessage: InAppMessage,
        callbacks: InAppMessagingDisplayCallbacks
    ) {
        _message.value = inAppMessage
        // Composeで表示
    }

    fun dismiss() {
        _message.value = null
    }
}
```

---

## カスタムバナー

```kotlin
@Composable
fun AppBanner(
    message: String?,
    type: BannerType = BannerType.INFO,
    onDismiss: () -> Unit
) {
    AnimatedVisibility(
        visible = message != null,
        enter = slideInVertically() + fadeIn(),
        exit = slideOutVertically() + fadeOut()
    ) {
        val backgroundColor = when (type) {
            BannerType.INFO -> MaterialTheme.colorScheme.primaryContainer
            BannerType.SUCCESS -> Color(0xFF4CAF50)
            BannerType.WARNING -> Color(0xFFFFC107)
            BannerType.ERROR -> MaterialTheme.colorScheme.errorContainer
        }

        Surface(
            color = backgroundColor,
            modifier = Modifier.fillMaxWidth()
        ) {
            Row(
                Modifier.padding(12.dp),
                verticalAlignment = Alignment.CenterVertically
            ) {
                Text(
                    message ?: "",
                    modifier = Modifier.weight(1f),
                    style = MaterialTheme.typography.bodyMedium
                )
                IconButton(onClick = onDismiss) {
                    Icon(Icons.Default.Close, "閉じる")
                }
            }
        }
    }
}

enum class BannerType { INFO, SUCCESS, WARNING, ERROR }
```

---

## SnackbarHost統合

```kotlin
@Composable
fun MessageScaffold(viewModel: MessageViewModel = hiltViewModel()) {
    val snackbarHostState = remember { SnackbarHostState() }
    val message by viewModel.message.collectAsStateWithLifecycle()

    LaunchedEffect(message) {
        message?.let {
            val result = snackbarHostState.showSnackbar(
                message = it.text,
                actionLabel = it.actionLabel,
                duration = SnackbarDuration.Short
            )
            if (result == SnackbarResult.ActionPerformed) {
                it.action?.invoke()
            }
            viewModel.clearMessage()
        }
    }

    Scaffold(snackbarHost = { SnackbarHost(snackbarHostState) }) { padding ->
        // コンテンツ
    }
}

data class AppMessage(
    val text: String,
    val actionLabel: String? = null,
    val action: (() -> Unit)? = null
)
```

---

## まとめ

| 手法 | 用途 |
|------|------|
| Firebase IAM | リモート配信メッセージ |
| カスタムバナー | アプリ内通知 |
| SnackbarHost | 一時的フィードバック |
| Dialog | 重要な確認 |

- Firebase In-App Messagingでリモート制御メッセージ
- `AnimatedVisibility`でバナーをスムーズ表示
- `SnackbarHost`でアクション付きフィードバック
- バナータイプで色分けしてUX向上

---

8種類のAndroidアプリテンプレート（メッセージUI対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Firebase Cloud Messaging](https://zenn.dev/myougatheaxo/articles/android-compose-firebase-cloud-messaging-2026)
- [通知](https://zenn.dev/myougatheaxo/articles/android-compose-notification-2026)
- [アニメーション](https://zenn.dev/myougatheaxo/articles/android-compose-animation-spring-2026)
