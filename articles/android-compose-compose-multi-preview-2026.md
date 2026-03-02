---
title: "MultiPreview完全ガイド — @PreviewParameter/カスタムプレビュー/ダークモード一括"
emoji: "👀"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "preview"]
published: true
---

## この記事で学べること

**MultiPreview**（@Preview、@PreviewParameter、カスタムマルチプレビュー、デバイス/テーマ一括プレビュー）を解説します。

---

## カスタムMultiPreview

```kotlin
@Preview(name = "Light", showBackground = true)
@Preview(name = "Dark", showBackground = true, uiMode = UI_MODE_NIGHT_YES)
@Preview(name = "Large Font", showBackground = true, fontScale = 1.5f)
annotation class ThemePreview

@ThemePreview
@Composable
fun UserCardPreview() {
    AppTheme {
        UserCard(name = "山田太郎", email = "taro@example.com")
    }
}
```

---

## @PreviewParameter

```kotlin
class UserPreviewProvider : PreviewParameterProvider<User> {
    override val values = sequenceOf(
        User(1, "山田太郎", "taro@example.com"),
        User(2, "田中花子", "hanako@example.com"),
        User(3, "", "")  // 空の状態
    )
}

@Preview(showBackground = true)
@Composable
fun UserCardParameterPreview(
    @PreviewParameter(UserPreviewProvider::class) user: User
) {
    AppTheme { UserCard(name = user.name, email = user.email) }
}
```

---

## デバイスプレビュー

```kotlin
@Preview(name = "Phone", device = Devices.PHONE, showSystemUi = true)
@Preview(name = "Tablet", device = Devices.TABLET, showSystemUi = true)
@Preview(name = "Foldable", device = Devices.FOLDABLE, showSystemUi = true)
annotation class DevicePreview

@DevicePreview
@Composable
fun MainScreenPreview() {
    AppTheme { MainScreen() }
}

// UiStateプレビュー
class UiStateProvider : PreviewParameterProvider<UiState> {
    override val values = sequenceOf(
        UiState.Loading,
        UiState.Success(listOf("Item 1", "Item 2")),
        UiState.Error("ネットワークエラー")
    )
}

@Preview(showBackground = true)
@Composable
fun ContentPreview(@PreviewParameter(UiStateProvider::class) state: UiState) {
    AppTheme {
        when (state) {
            is UiState.Loading -> CircularProgressIndicator()
            is UiState.Success -> LazyColumn { items(state.items) { Text(it) } }
            is UiState.Error -> Text(state.message, color = Color.Red)
        }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `@Preview` | プレビュー定義 |
| `@PreviewParameter` | パラメータ注入 |
| カスタムAnnotation | マルチプレビュー |
| `Devices.*` | デバイスプレビュー |

- カスタムAnnotationで複数プレビューを一括定義
- `@PreviewParameter`でデータバリエーション
- `showSystemUi`でシステムUI付きプレビュー
- Light/Dark/LargeFont/デバイスの全パターン確認

---

8種類のAndroidアプリテンプレート（プレビュー付き）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Screenshot Test](https://zenn.dev/myougatheaxo/articles/android-compose-compose-screenshot-test-2026)
- [UIテスト](https://zenn.dev/myougatheaxo/articles/android-compose-compose-ui-test-2026)
- [Adaptive Layout](https://zenn.dev/myougatheaxo/articles/android-compose-compose-adaptive-layout-2026)
