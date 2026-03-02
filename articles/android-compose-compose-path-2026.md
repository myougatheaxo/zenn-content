---
title: "Path描画完全ガイド — カスタムシェイプ/ベジェ曲線/clipPath"
emoji: "✏️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "canvas"]
published: true
---

## この記事で学べること

**Path描画**（カスタムシェイプ、ベジェ曲線、clipPath、PathEffect）を解説します。

---

## 基本Path

```kotlin
@Composable
fun BasicPathDemo() {
    Canvas(Modifier.size(200.dp)) {
        val path = Path().apply {
            moveTo(size.width / 2, 0f)
            lineTo(size.width, size.height * 0.4f)
            lineTo(size.width * 0.8f, size.height)
            lineTo(size.width * 0.2f, size.height)
            lineTo(0f, size.height * 0.4f)
            close()
        }
        drawPath(path, Color(0xFF42A5F5))
    }
}
```

---

## ベジェ曲線

```kotlin
@Composable
fun BezierCurveDemo() {
    Canvas(Modifier.fillMaxWidth().height(200.dp)) {
        val path = Path().apply {
            moveTo(0f, size.height)
            // 二次ベジェ
            quadraticBezierTo(
                size.width * 0.25f, 0f,
                size.width * 0.5f, size.height * 0.5f
            )
            // 三次ベジェ
            cubicTo(
                size.width * 0.75f, size.height,
                size.width * 0.85f, 0f,
                size.width, size.height * 0.3f
            )
        }
        drawPath(
            path, Color(0xFFE91E63),
            style = Stroke(width = 3.dp.toPx(), cap = StrokeCap.Round)
        )
    }
}

// 波形背景
@Composable
fun WaveBackground() {
    Canvas(Modifier.fillMaxWidth().height(150.dp)) {
        val wavePath = Path().apply {
            moveTo(0f, size.height * 0.5f)
            val waveLength = size.width / 3
            for (i in 0..3) {
                quadraticBezierTo(
                    waveLength * i + waveLength * 0.25f, size.height * 0.2f,
                    waveLength * i + waveLength * 0.5f, size.height * 0.5f
                )
                quadraticBezierTo(
                    waveLength * i + waveLength * 0.75f, size.height * 0.8f,
                    waveLength * (i + 1), size.height * 0.5f
                )
            }
            lineTo(size.width, size.height)
            lineTo(0f, size.height)
            close()
        }
        drawPath(wavePath, Brush.verticalGradient(listOf(Color(0xFF42A5F5), Color(0xFF1976D2))))
    }
}
```

---

## clipPath

```kotlin
@Composable
fun ClipPathDemo() {
    Canvas(Modifier.size(200.dp)) {
        val heartPath = Path().apply {
            val w = size.width; val h = size.height
            moveTo(w / 2, h * 0.3f)
            cubicTo(w * 0.1f, -h * 0.1f, -w * 0.1f, h * 0.5f, w / 2, h)
            moveTo(w / 2, h * 0.3f)
            cubicTo(w * 0.9f, -h * 0.1f, w * 1.1f, h * 0.5f, w / 2, h)
        }
        clipPath(heartPath) {
            drawRect(Brush.linearGradient(listOf(Color.Red, Color(0xFFFF69B4))))
        }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `Path` | カスタムシェイプ定義 |
| `quadraticBezierTo` | 二次ベジェ曲線 |
| `cubicTo` | 三次ベジェ曲線 |
| `clipPath` | パスでクリッピング |

- `Path`でカスタムシェイプを自由に定義
- ベジェ曲線で滑らかな曲線描画
- `clipPath`でパス形状に切り抜き
- `Stroke`/`Fill`で描画スタイル変更

---

8種類のAndroidアプリテンプレート（カスタムUI対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [カスタムCanvas](https://zenn.dev/myougatheaxo/articles/android-compose-custom-canvas-2026)
- [Brush](https://zenn.dev/myougatheaxo/articles/android-compose-compose-brush-2026)
- [drawWithCache](https://zenn.dev/myougatheaxo/articles/android-compose-compose-draw-with-cache-2026)
