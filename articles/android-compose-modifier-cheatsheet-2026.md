---
title: "Compose Modifierチートシート — 全パターン一覧"
emoji: "📋"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "modifier"]
published: true
---

## この記事で学べること

Composeの**Modifier**を全カテゴリ（サイズ、配置、装飾、入力）でまとめます。

---

## サイズ

```kotlin
Modifier
    .fillMaxSize()              // 親いっぱい
    .fillMaxWidth()             // 幅いっぱい
    .fillMaxHeight()            // 高さいっぱい
    .fillMaxWidth(0.5f)         // 幅の50%
    .size(100.dp)               // 幅高さ同じ
    .size(width = 200.dp, height = 100.dp)
    .width(200.dp)
    .height(100.dp)
    .widthIn(min = 100.dp, max = 300.dp)  // 最小最大幅
    .heightIn(min = 50.dp)
    .wrapContentSize()          // コンテンツサイズ
    .aspectRatio(16f / 9f)      // アスペクト比
    .weight(1f)                 // Row/Column内の比率
```

---

## 余白

```kotlin
Modifier
    .padding(16.dp)                        // 全方向
    .padding(horizontal = 16.dp, vertical = 8.dp)
    .padding(start = 16.dp, top = 8.dp, end = 16.dp, bottom = 8.dp)
    .offset(x = 10.dp, y = 5.dp)          // オフセット（レイアウト影響なし）
    .offset { IntOffset(x, y) }            // 動的オフセット
```

---

## 装飾

```kotlin
Modifier
    .background(Color.Blue)
    .background(Color.Blue, RoundedCornerShape(12.dp))
    .background(Brush.linearGradient(listOf(Color.Red, Color.Blue)))
    .border(2.dp, Color.Gray)
    .border(2.dp, Color.Gray, CircleShape)
    .clip(RoundedCornerShape(12.dp))
    .clip(CircleShape)
    .shadow(8.dp, RoundedCornerShape(12.dp))
    .alpha(0.5f)
    .rotate(45f)
    .scale(1.5f)
```

---

## クリック/入力

```kotlin
Modifier
    .clickable { /* タップ */ }
    .clickable(indication = null, interactionSource = remember { MutableInteractionSource() }) {}
    .toggleable(value = checked, onValueChange = { checked = it })
    .selectable(selected = selected, onClick = {})
    .combinedClickable(
        onClick = { /* タップ */ },
        onLongClick = { /* 長押し */ },
        onDoubleClick = { /* ダブルタップ */ }
    )
    .scrollable(rememberScrollableState { delta -> delta }, Orientation.Vertical)
    .draggable(rememberDraggableState { delta -> }, Orientation.Horizontal)
    .focusable()
```

---

## スクロール

```kotlin
// 縦スクロール
Column(Modifier.verticalScroll(rememberScrollState())) {
    // コンテンツ
}

// 横スクロール
Row(Modifier.horizontalScroll(rememberScrollState())) {
    // コンテンツ
}

// nestedScroll
Modifier.nestedScroll(connection)
```

---

## 描画/レイアウト

```kotlin
Modifier
    .graphicsLayer {
        scaleX = 1.5f
        scaleY = 1.5f
        rotationZ = 45f
        translationX = 100f
        alpha = 0.8f
    }
    .drawBehind { drawRect(Color.Red) }     // 背面に描画
    .drawWithContent {
        drawContent()                        // コンテンツ描画
        drawCircle(Color.Red, 10f)          // 前面に描画
    }
    .zIndex(1f)                             // Z軸の順序
    .testTag("my_tag")                      // テスト用タグ
    .semantics { contentDescription = "説明" }
```

---

## 順序の重要性

```kotlin
// ❌ paddingの後にbackground → paddingが背景色に含まれない
Modifier.padding(16.dp).background(Color.Blue)

// ✅ backgroundの後にpadding → 背景色が全体に適用
Modifier.background(Color.Blue).padding(16.dp)

// ❌ clipの前にshadow → 影がクリップされる
Modifier.clip(CircleShape).shadow(8.dp)

// ✅ shadowの後にclip
Modifier.shadow(8.dp, CircleShape).clip(CircleShape)
```

---

## まとめ

- Modifierは**順序が重要**（上から下へ適用）
- `padding`と`background`の順序で見た目が変わる
- `fillMaxSize`/`weight`でレイアウト制御
- `clickable`/`combinedClickable`で入力処理
- `graphicsLayer`でパフォーマンスの良い変形
- `testTag`/`semantics`でテスト/アクセシビリティ

---

8種類のAndroidアプリテンプレート（Modifier設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [カスタムレイアウト](https://zenn.dev/myougatheaxo/articles/android-compose-custom-layout-2026)
- [Canvas描画](https://zenn.dev/myougatheaxo/articles/android-compose-canvas-drawing-2026)
- [ジェスチャー検出](https://zenn.dev/myougatheaxo/articles/android-compose-gesture-detection-2026)
