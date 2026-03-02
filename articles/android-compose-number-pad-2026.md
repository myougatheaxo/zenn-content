---
title: "数字パッド/PIN入力完全ガイド — カスタムキーパッド/セキュリティ/生体認証連携"
emoji: "🔢"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "ui"]
published: true
---

## この記事で学べること

**数字パッド/PIN入力**（カスタムキーパッド、ドット表示、シャッフル、生体認証連携）を解説します。

---

## PIN入力画面

```kotlin
@Composable
fun PinInputScreen(
    pinLength: Int = 4,
    onPinComplete: (String) -> Unit
) {
    var pin by remember { mutableStateOf("") }
    var isError by remember { mutableStateOf(false) }
    val haptic = LocalHapticFeedback.current

    Column(
        Modifier.fillMaxSize().padding(32.dp),
        horizontalAlignment = Alignment.CenterHorizontally,
        verticalArrangement = Arrangement.Center
    ) {
        Text("PINコードを入力", style = MaterialTheme.typography.headlineSmall)
        Spacer(Modifier.height(32.dp))

        // ドット表示
        PinDots(
            pinLength = pinLength,
            filledCount = pin.length,
            isError = isError
        )

        Spacer(Modifier.height(48.dp))

        // 数字パッド
        NumberPad(
            onNumberClick = { number ->
                if (pin.length < pinLength) {
                    pin += number
                    haptic.performHapticFeedback(HapticFeedbackType.TextHandleMove)
                    if (pin.length == pinLength) {
                        onPinComplete(pin)
                    }
                }
            },
            onDeleteClick = {
                if (pin.isNotEmpty()) {
                    pin = pin.dropLast(1)
                    isError = false
                }
            }
        )
    }
}
```

---

## ドット表示

```kotlin
@Composable
fun PinDots(pinLength: Int, filledCount: Int, isError: Boolean) {
    val shakeOffset = remember { Animatable(0f) }
    val scope = rememberCoroutineScope()

    LaunchedEffect(isError) {
        if (isError) {
            // シェイクアニメーション
            repeat(3) {
                shakeOffset.animateTo(10f, tween(50))
                shakeOffset.animateTo(-10f, tween(50))
            }
            shakeOffset.animateTo(0f, tween(50))
        }
    }

    Row(
        Modifier.offset { IntOffset(shakeOffset.value.toInt(), 0) },
        horizontalArrangement = Arrangement.spacedBy(16.dp)
    ) {
        repeat(pinLength) { index ->
            val isFilled = index < filledCount
            val scale by animateFloatAsState(
                targetValue = if (isFilled) 1.2f else 1f,
                animationSpec = spring(dampingRatio = Spring.DampingRatioMediumBouncy),
                label = "dotScale"
            )

            Box(
                Modifier
                    .size(16.dp)
                    .graphicsLayer { scaleX = scale; scaleY = scale }
                    .background(
                        if (isError) MaterialTheme.colorScheme.error
                        else if (isFilled) MaterialTheme.colorScheme.primary
                        else MaterialTheme.colorScheme.outlineVariant,
                        CircleShape
                    )
            )
        }
    }
}
```

---

## 数字パッド

```kotlin
@Composable
fun NumberPad(
    onNumberClick: (String) -> Unit,
    onDeleteClick: () -> Unit
) {
    val keys = listOf(
        listOf("1", "2", "3"),
        listOf("4", "5", "6"),
        listOf("7", "8", "9"),
        listOf("", "0", "⌫")
    )

    Column(horizontalAlignment = Alignment.CenterHorizontally) {
        keys.forEach { row ->
            Row(horizontalArrangement = Arrangement.spacedBy(24.dp)) {
                row.forEach { key ->
                    if (key.isEmpty()) {
                        Spacer(Modifier.size(72.dp))
                    } else {
                        Box(
                            Modifier
                                .size(72.dp)
                                .clip(CircleShape)
                                .clickable {
                                    if (key == "⌫") onDeleteClick()
                                    else onNumberClick(key)
                                },
                            contentAlignment = Alignment.Center
                        ) {
                            if (key == "⌫") {
                                Icon(Icons.AutoMirrored.Filled.Backspace, "削除", Modifier.size(24.dp))
                            } else {
                                Text(key, fontSize = 28.sp, fontWeight = FontWeight.Light)
                            }
                        }
                    }
                }
            }
            Spacer(Modifier.height(12.dp))
        }
    }
}
```

---

## まとめ

| 機能 | 実装 |
|------|------|
| ドット表示 | `CircleShape` + `animateFloat` |
| 数字パッド | Grid配置 + `clickable` |
| エラー表示 | シェイクアニメーション |
| ハプティクス | `HapticFeedbackType` |

- カスタム数字パッドでPIN入力UI
- ドットのバウンスアニメーションで入力フィードバック
- エラー時のシェイクで視覚的に通知
- ハプティクスで触覚フィードバック

---

8種類のAndroidアプリテンプレート（認証UI実装済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [生体認証](https://zenn.dev/myougatheaxo/articles/android-compose-biometric-auth-2026)
- [Springアニメーション](https://zenn.dev/myougatheaxo/articles/android-compose-animation-spring-2026)
- [Credential Manager](https://zenn.dev/myougatheaxo/articles/android-compose-credential-manager-2026)
