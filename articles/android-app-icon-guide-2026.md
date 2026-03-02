---
title: "Androidアプリアイコンの作り方 — Adaptive Iconを5分で完全設定"
emoji: "🎯"
type: "tech"
topics: ["android", "design", "androidstudio", "icon"]
published: true
---

## この記事で学べること

アプリのアイコンは第一印象を決めます。Android 8.0以降の**Adaptive Icon**の仕組みと、Android Studioで5分で設定する方法を解説します。

---

## Adaptive Iconとは

Adaptive Iconは**前景（foreground）と背景（background）の2レイヤー**で構成されます。

```
┌─────────────────┐
│   背景レイヤー     │ ← 単色 or グラデーション
│  ┌───────────┐  │
│  │ 前景レイヤー │  │ ← ロゴ・シンボル
│  └───────────┘  │
└─────────────────┘
```

OSやランチャーがアイコンの形（丸、角丸四角、しずく型など）を決定。開発者は中身だけ用意すれば、全ての形に自動対応します。

---

## Image Asset Studioで作成

### Step 1: 起動

Android Studio → `res`フォルダ右クリック → **New** → **Image Asset**

### Step 2: 前景レイヤー

- **Source Asset**: ロゴ画像（PNG推奨、1024x1024px）
- **Trim**: 余白を自動トリム
- **Resize**: セーフゾーン内に収まるよう調整（66%が目安）

### Step 3: 背景レイヤー

- **Color**: 単色（ブランドカラー推奨）
- **Image**: グラデーション画像も指定可能

### Step 4: 生成

**Next** → **Finish** で以下が自動生成：

```
res/
├── mipmap-mdpi/      (48x48)
├── mipmap-hdpi/      (72x72)
├── mipmap-xhdpi/     (96x96)
├── mipmap-xxhdpi/    (144x144)
├── mipmap-xxxhdpi/   (192x192)
└── mipmap-anydpi-v26/
    ├── ic_launcher.xml        (Adaptive Icon定義)
    └── ic_launcher_round.xml  (丸型)
```

---

## ic_launcher.xml の構造

```xml
<?xml version="1.0" encoding="utf-8"?>
<adaptive-icon xmlns:android="http://schemas.android.com/apk/res/android">
    <background android:drawable="@color/ic_launcher_background"/>
    <foreground android:drawable="@drawable/ic_launcher_foreground"/>
    <monochrome android:drawable="@drawable/ic_launcher_foreground"/>
</adaptive-icon>
```

---

## Android 13+ モノクロアイコン

Android 13では、ユーザーがテーマアイコンを有効にすると、アイコンがモノクロ表示されます。

```xml
<monochrome android:drawable="@drawable/ic_launcher_monochrome"/>
```

モノクロレイヤーを指定しないと、テーマアイコン有効時にアプリアイコンだけ浮いて見えます。

---

## デザインTips

| ポイント | 推奨 |
|---------|------|
| セーフゾーン | 中央66%以内にロゴを配置 |
| 背景色 | ブランドカラー1色 |
| 前景 | シンプルなシルエット |
| テキスト | 入れない（小さくて読めない） |
| 複雑な形 | 避ける（丸型で切れる） |

---

## まとめ

- Adaptive Icon = 前景 + 背景の2レイヤー
- Image Asset Studioで5分で全解像度生成
- Android 13+のモノクロアイコンも設定推奨
- セーフゾーン（66%）内にロゴを収める

---

8種類のAndroidアプリテンプレート（アイコン設定済み、カスタマイズ手順付き）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Google Play公開チェックリスト](https://zenn.dev/myougatheaxo/articles/google-play-publish-guide-2026)
- [Material3テーマ完全ガイド](https://zenn.dev/myougatheaxo/articles/material3-theme-guide-2026)
- [APKサイズを半分にする5つのテクニック](https://zenn.dev/myougatheaxo/articles/android-apk-optimization-2026)
