---
title: "Accompanist完全ガイド — Permissions/SystemUiController/WebView/Placeholder"
emoji: "🎵"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "accompanist"]
published: true
---

## この記事で学べること

**Accompanist**（Permissions、SystemUiController、WebView、Placeholder）を解説します。※多くの機能は公式Composeに統合済み。

---

## Accompanist Permissions

```kotlin
// build.gradle.kts
dependencies {
    implementation("com.google.accompanist:accompanist-permissions:0.36.0")
}

@OptIn(ExperimentalPermissionsApi::class)
@Composable
fun CameraPermissionScreen() {
    val cameraPermission = rememberPermissionState(Manifest.permission.CAMERA)

    when {
        cameraPermission.status.isGranted -> {
            CameraPreview()
        }
        cameraPermission.status.shouldShowRationale -> {
            Column(Modifier.padding(16.dp)) {
                Text("カメラの使用許可が必要です")
                Text("写真撮影機能にはカメラへのアクセスが必要です。")
                Button(onClick = { cameraPermission.launchPermissionRequest() }) {
                    Text("許可する")
                }
            }
        }
        else -> {
            Button(onClick = { cameraPermission.launchPermissionRequest() }) {
                Text("カメラを許可")
            }
        }
    }
}

// 複数パーミッション
@OptIn(ExperimentalPermissionsApi::class)
@Composable
fun MultiplePermissionsExample() {
    val permissions = rememberMultiplePermissionsState(
        listOf(Manifest.permission.CAMERA, Manifest.permission.RECORD_AUDIO)
    )

    if (permissions.allPermissionsGranted) {
        VideoRecorder()
    } else {
        Button(onClick = { permissions.launchMultiplePermissionRequest() }) {
            Text("カメラとマイクを許可")
        }
    }
}
```

---

## 公式Composeへの移行マップ

```kotlin
// ❌ Accompanist SystemUiController（非推奨）
// SystemUiController was deprecated
// ✅ 公式: enableEdgeToEdge() in Activity

// ❌ Accompanist Pager（非推奨）
// ✅ 公式: HorizontalPager / VerticalPager

// ❌ Accompanist SwipeRefresh（非推奨）
// ✅ 公式: PullToRefreshBox

// ❌ Accompanist Navigation Animation（非推奨）
// ✅ 公式: AnimatedNavHost

// ❌ Accompanist FlowLayout（非推奨）
// ✅ 公式: FlowRow / FlowColumn

// ✅ まだAccompanistが必要
// - Permissions (rememberPermissionState)
// - Placeholder (shimmer)
// - Adaptive (一部)
```

---

## Placeholder

```kotlin
@Composable
fun LoadingCard(isLoading: Boolean) {
    Card(Modifier.fillMaxWidth().padding(8.dp)) {
        Row(Modifier.padding(16.dp)) {
            Box(
                Modifier
                    .size(48.dp)
                    .clip(CircleShape)
                    .placeholder(
                        visible = isLoading,
                        highlight = PlaceholderHighlight.shimmer()
                    )
            )
            Column(Modifier.padding(start = 16.dp)) {
                Text(
                    "ユーザー名",
                    modifier = Modifier.placeholder(
                        visible = isLoading,
                        highlight = PlaceholderHighlight.shimmer()
                    )
                )
                Text(
                    "説明文テキスト",
                    modifier = Modifier.placeholder(
                        visible = isLoading,
                        highlight = PlaceholderHighlight.fade()
                    )
                )
            }
        }
    }
}
```

---

## まとめ

| ライブラリ | 状態 |
|-----------|------|
| Permissions | 現役（公式未統合） |
| Placeholder | 現役 |
| Pager | 非推奨→公式 |
| SwipeRefresh | 非推奨→公式 |
| SystemUiController | 非推奨→公式 |

- Accompanist Permissionsは公式未統合で現役
- Pager/SwipeRefresh/FlowLayoutは公式Composeに移行
- `placeholder` + `shimmer()`でローディングUI
- 新規プロジェクトは可能な限り公式APIを使用

---

8種類のAndroidアプリテンプレート（最新API対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [パーミッション](https://zenn.dev/myougatheaxo/articles/android-compose-permission-2026)
- [Shimmer/Placeholder](https://zenn.dev/myougatheaxo/articles/android-compose-shimmer-placeholder-2026)
- [Edge-to-Edge](https://zenn.dev/myougatheaxo/articles/android-compose-edge-to-edge-2026)
