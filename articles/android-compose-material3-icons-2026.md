---
title: "Material Icons完全ガイド — アイコン選定/カスタムアイコン/ベクター"
emoji: "🎨"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "material3"]
published: true
---

## この記事で学べること

Composeでの**Material Icons**の使い方（アイコン選定、カスタムアイコン、ベクタードローアブル）を解説します。

---

## 基本のアイコン

```kotlin
// Material Icons (デフォルト)
Icon(Icons.Default.Home, contentDescription = "ホーム")
Icon(Icons.Default.Search, contentDescription = "検索")
Icon(Icons.Default.Settings, contentDescription = "設定")

// サイズとカラー変更
Icon(
    Icons.Default.Favorite,
    contentDescription = "お気に入り",
    modifier = Modifier.size(32.dp),
    tint = Color.Red
)
```

---

## アイコンスタイル

```kotlin
// 5つのスタイル
Icon(Icons.Filled.Star, null)     // 塗りつぶし（Default）
Icon(Icons.Outlined.Star, null)   // アウトライン
Icon(Icons.Rounded.Star, null)    // 角丸
Icon(Icons.Sharp.Star, null)      // シャープ
Icon(Icons.TwoTone.Star, null)    // ツートン

// 拡張アイコン（別途依存関係追加）
// implementation("androidx.compose.material:material-icons-extended")
Icon(Icons.Outlined.Analytics, null)
Icon(Icons.Outlined.QrCode, null)
```

---

## アイコンボタン

```kotlin
@Composable
fun IconButtonExamples() {
    Row(horizontalArrangement = Arrangement.spacedBy(8.dp)) {
        // 標準
        IconButton(onClick = { }) {
            Icon(Icons.Default.Edit, "編集")
        }

        // Filled
        FilledIconButton(onClick = { }) {
            Icon(Icons.Default.Add, "追加")
        }

        // Tonal
        FilledTonalIconButton(onClick = { }) {
            Icon(Icons.Default.Share, "共有")
        }

        // Outlined
        OutlinedIconButton(onClick = { }) {
            Icon(Icons.Default.Delete, "削除")
        }
    }
}
```

---

## トグルアイコンボタン

```kotlin
@Composable
fun FavoriteToggle(
    isFavorite: Boolean,
    onToggle: (Boolean) -> Unit
) {
    IconToggleButton(
        checked = isFavorite,
        onCheckedChange = onToggle
    ) {
        Icon(
            if (isFavorite) Icons.Filled.Favorite else Icons.Outlined.FavoriteBorder,
            contentDescription = "お気に入り",
            tint = if (isFavorite) Color.Red else MaterialTheme.colorScheme.onSurface
        )
    }
}
```

---

## ベクタードローアブル

```kotlin
// res/drawable/ic_custom.xml を使用
Icon(
    painter = painterResource(R.drawable.ic_custom),
    contentDescription = "カスタムアイコン",
    modifier = Modifier.size(24.dp),
    tint = MaterialTheme.colorScheme.primary
)

// tint無効化（元の色を使用）
Icon(
    painter = painterResource(R.drawable.ic_logo),
    contentDescription = "ロゴ",
    tint = Color.Unspecified
)
```

---

## アイコンとテキストの組み合わせ

```kotlin
@Composable
fun IconWithLabel(
    icon: ImageVector,
    label: String,
    onClick: () -> Unit
) {
    Column(
        horizontalAlignment = Alignment.CenterHorizontally,
        modifier = Modifier
            .clickable(onClick = onClick)
            .padding(8.dp)
    ) {
        Icon(
            icon,
            contentDescription = label,
            modifier = Modifier.size(28.dp),
            tint = MaterialTheme.colorScheme.primary
        )
        Spacer(Modifier.height(4.dp))
        Text(
            label,
            style = MaterialTheme.typography.labelSmall
        )
    }
}

// グリッドメニュー
@Composable
fun IconGrid() {
    val items = listOf(
        Icons.Default.Camera to "カメラ",
        Icons.Default.Photo to "写真",
        Icons.Default.VideoCall to "動画",
        Icons.Default.MusicNote to "音楽",
        Icons.Default.Map to "地図",
        Icons.Default.CalendarMonth to "カレンダー"
    )

    LazyVerticalGrid(columns = GridCells.Fixed(3)) {
        items(items) { (icon, label) ->
            IconWithLabel(icon = icon, label = label, onClick = { })
        }
    }
}
```

---

## まとめ

- `Icons.Default.*`（= `Icons.Filled.*`）がデフォルト
- 5スタイル: Filled, Outlined, Rounded, Sharp, TwoTone
- `material-icons-extended`で全アイコン利用可能
- `FilledIconButton`/`OutlinedIconButton`でボタンバリエーション
- `IconToggleButton`でトグル（お気に入り等）
- `painterResource`でカスタムベクタードローアブル

---

8種類のAndroidアプリテンプレート（Material Icons設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Material3コンポーネント一覧](https://zenn.dev/myougatheaxo/articles/compose-material3-components-2026)
- [ボタンバリエーション](https://zenn.dev/myougatheaxo/articles/android-compose-button-variants-2026)
- [カスタムテーマガイド](https://zenn.dev/myougatheaxo/articles/android-compose-theme-custom-2026)
