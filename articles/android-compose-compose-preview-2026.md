---
title: "Compose Preview完全ガイド — @Preview/マルチプレビュー/インタラクティブ/デバイス指定"
emoji: "👁️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "compose"]
published: true
---

## この記事で学べること

**Compose Preview**（@Preview、マルチプレビュー、デバイス指定、ダークモード、インタラクティブモード）を解説します。

---

## @Preview基本

```kotlin
@Preview(
    name = "Light Mode",
    showBackground = true,
    backgroundColor = 0xFFFFFFFF
)
@Preview(
    name = "Dark Mode",
    showBackground = true,
    backgroundColor = 0xFF121212,
    uiMode = Configuration.UI_MODE_NIGHT_YES
)
@Composable
fun UserCardPreview() {
    MaterialTheme {
        UserCard(
            user = User(id = 1, name = "テスト太郎", email = "test@example.com"),
            onClick = {}
        )
    }
}
```

---

## マルチプレビューアノテーション

```kotlin
@Preview(name = "Phone", device = Devices.PHONE, showSystemUi = true)
@Preview(name = "Tablet", device = Devices.TABLET, showSystemUi = true)
@Preview(name = "Foldable", device = Devices.FOLDABLE, showSystemUi = true)
annotation class DevicePreviews

@Preview(name = "Small", fontScale = 0.85f)
@Preview(name = "Normal", fontScale = 1.0f)
@Preview(name = "Large", fontScale = 1.15f)
@Preview(name = "Extra Large", fontScale = 1.3f)
annotation class FontScalePreviews

@DevicePreviews
@FontScalePreviews
@Composable
fun HomeScreenPreview() {
    MaterialTheme {
        HomeScreen()
    }
}
```

---

## PreviewParameterProvider

```kotlin
class UserPreviewProvider : PreviewParameterProvider<User> {
    override val values = sequenceOf(
        User(1, "短い名前", "a@b.com"),
        User(2, "とても長いユーザー名のテストケース", "longname@example.com"),
        User(3, "", "noname@example.com")
    )
}

@Preview(showBackground = true)
@Composable
fun UserCardVariants(
    @PreviewParameter(UserPreviewProvider::class) user: User
) {
    MaterialTheme {
        UserCard(user = user, onClick = {})
    }
}
```

---

## カスタムデバイス

```kotlin
@Preview(
    name = "Custom Device",
    device = "spec:width=411dp,height=891dp,dpi=420,isRound=false,chinSize=0dp,orientation=landscape"
)
@Composable
fun LandscapePreview() {
    MaterialTheme {
        LandscapeLayout()
    }
}

// ロケール指定
@Preview(locale = "ja")
@Preview(locale = "en")
@Preview(locale = "zh")
@Composable
fun LocalizedPreview() {
    MaterialTheme {
        SettingsScreen()
    }
}
```

---

## まとめ

| パラメータ | 用途 |
|-----------|------|
| `showBackground` | 背景表示 |
| `device` | デバイス指定 |
| `uiMode` | ダークモード |
| `fontScale` | フォントサイズ |
| `locale` | 言語 |

- `@Preview`で即座にUIを確認
- マルチプレビューアノテーションで一括プレビュー
- `PreviewParameterProvider`でデータバリエーション
- デバイス指定でタブレット/折りたたみ確認

---

8種類のAndroidアプリテンプレート（Preview設定済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose Testing](https://zenn.dev/myougatheaxo/articles/android-compose-compose-testing-2026)
- [スクリーンショットテスト](https://zenn.dev/myougatheaxo/articles/android-compose-screenshot-test-2026)
- [Material3テーマ](https://zenn.dev/myougatheaxo/articles/android-compose-material3-theme-2026)
