---
title: "アクセシビリティ対応入門 — contentDescriptionだけで差がつくアプリ品質"
emoji: "♿"
type: "tech"
topics: ["android", "jetpackcompose", "accessibility", "kotlin"]
published: true
---

## この記事で学べること

Google Playの審査では、アクセシビリティ対応が年々重要視されています。AIが生成するコードは基本的にMaterial3を使っているため最低限の対応はされていますが、**contentDescriptionの追加**など手動で行うべき箇所があります。

---

## 最低限やるべき3つ

### 1. contentDescription（画像・アイコン）

```kotlin
// ❌ 未設定
Icon(Icons.Default.Delete, contentDescription = null)

// ✅ 設定済み
Icon(Icons.Default.Delete, contentDescription = "削除")
```

スクリーンリーダー（TalkBack）がアイコンを読み上げる際に使われます。`null`だと無視される。

**装飾的な画像**（意味を持たない背景など）は`null`でOK。情報を持つ画像には必ず設定。

### 2. タッチターゲット（48dp以上）

```kotlin
// ❌ 小さすぎる
IconButton(
    onClick = { /* ... */ },
    modifier = Modifier.size(24.dp)  // タップしにくい
) {
    Icon(Icons.Default.Close, "閉じる")
}

// ✅ Material3のIconButtonはデフォルト48dp
IconButton(onClick = { /* ... */ }) {
    Icon(Icons.Default.Close, "閉じる")
}
```

Material3の`IconButton`はデフォルトで48dpのタッチ領域を確保。カスタムボタンを作る場合は`Modifier.defaultMinSize(minWidth = 48.dp, minHeight = 48.dp)`を追加。

### 3. 色のコントラスト比

| 要素 | 最低コントラスト比 |
|------|-----------------|
| 通常テキスト | 4.5:1 |
| 大テキスト（18sp以上） | 3:1 |
| UIコンポーネント | 3:1 |

Material3のデフォルトカラーは基準を満たしています。カスタムカラーを使う場合は[WebAIM Contrast Checker](https://webaim.org/resources/contrastchecker/)で確認。

---

## Composeのsemantics

```kotlin
@Composable
fun HabitItem(habit: Habit, onToggle: () -> Unit) {
    Row(
        modifier = Modifier
            .semantics {
                contentDescription = "${habit.name}、${if (habit.completed) "完了" else "未完了"}"
                stateDescription = if (habit.completed) "完了" else "未完了"
            }
            .clickable { onToggle() }
    ) {
        Checkbox(
            checked = habit.completed,
            onCheckedChange = null // Row全体がクリック可能
        )
        Text(habit.name)
    }
}
```

`semantics`ブロックでTalkBackへの情報提供を細かく制御できます。

---

## TalkBackテスト

1. **設定** → **ユーザー補助** → **TalkBack** をON
2. 画面を指でなぞると、要素が読み上げられる
3. contentDescriptionが`null`の画像は無視される

**テストすべき箇所**：
- 全てのアイコンが読み上げられるか
- ボタンの操作説明が適切か
- リストの各項目が区別できるか

---

## まとめ

- `contentDescription`は情報を持つ画像に必ず設定
- タッチターゲットは48dp以上（Material3デフォルトで達成）
- 色のコントラスト比はツールで確認
- TalkBackで実機テスト

---

8種類のAndroidアプリテンプレート（Material3準拠のアクセシビリティ対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Google Play公開チェックリスト](https://zenn.dev/myougatheaxo/articles/google-play-publish-guide-2026)
- [Material3テーマ完全ガイド](https://zenn.dev/myougatheaxo/articles/material3-theme-guide-2026)
- [MVVM設計の全体像](https://zenn.dev/myougatheaxo/articles/android-mvvm-architecture-2026)
