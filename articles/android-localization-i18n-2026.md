---
title: "Androidアプリの多言語対応（i18n）完全ガイド — strings.xmlからComposeまで"
emoji: "🌍"
type: "tech"
topics: ["android", "kotlin", "i18n", "localization"]
published: true
---

## この記事で学べること

日本語だけでなく英語や多言語に対応すれば、**世界中のユーザー**にリーチできます。Androidの多言語対応の正しい実装方法を解説します。

---

## strings.xml（基本）

```xml
<!-- res/values/strings.xml（デフォルト: 日本語） -->
<resources>
    <string name="app_name">習慣トラッカー</string>
    <string name="add_habit">習慣を追加</string>
    <string name="delete_confirm">%1$sを削除しますか？</string>
    <string name="streak_days">%1$d日連続</string>
</resources>
```

```xml
<!-- res/values-en/strings.xml（英語） -->
<resources>
    <string name="app_name">Habit Tracker</string>
    <string name="add_habit">Add Habit</string>
    <string name="delete_confirm">Delete %1$s?</string>
    <string name="streak_days">%1$d day streak</string>
</resources>
```

---

## Composeでの使い方

```kotlin
@Composable
fun HabitScreen() {
    Text(stringResource(R.string.app_name))

    // パラメータ付き
    Text(stringResource(R.string.delete_confirm, "読書"))
    Text(stringResource(R.string.streak_days, 7))
}
```

---

## 複数形（Plurals）

```xml
<resources>
    <plurals name="items_count">
        <item quantity="one">%1$d item</item>
        <item quantity="other">%1$d items</item>
    </plurals>
</resources>
```

```kotlin
val count = 5
Text(pluralStringResource(R.plurals.items_count, count, count))
```

---

## ディレクトリ構成

```
res/
├── values/              # デフォルト（日本語）
│   └── strings.xml
├── values-en/           # 英語
│   └── strings.xml
├── values-zh/           # 中国語（簡体字）
│   └── strings.xml
├── values-ko/           # 韓国語
│   └── strings.xml
└── values-es/           # スペイン語
    └── strings.xml
```

---

## RTL（右から左）対応

```xml
<!-- AndroidManifest.xml -->
<application
    android:supportsRtl="true">
```

```kotlin
// Composableでの注意
Row(
    modifier = Modifier.fillMaxWidth(),
    horizontalArrangement = Arrangement.Start  // RTL時に自動で右寄せ
) {
    Icon(Icons.Default.ArrowForward, null)  // RTL時にミラーリング
    Text(stringResource(R.string.next))
}
```

---

## 日付・数値のフォーマット

```kotlin
// 日付
val dateFormatter = DateTimeFormatter
    .ofLocalizedDate(FormatStyle.MEDIUM)
    .withLocale(Locale.getDefault())

Text(LocalDate.now().format(dateFormatter))

// 通貨
val currencyFormatter = NumberFormat.getCurrencyInstance(Locale.getDefault())
Text(currencyFormatter.format(1234.56))
// 日本: ¥1,235
// 米国: $1,234.56
```

---

## アプリ内言語切り替え（Android 13+）

```kotlin
// Android 13+ Per-App Language Preferences
val appLocale = LocaleListCompat.forLanguageTags("en")
AppCompatDelegate.setApplicationLocales(appLocale)
```

```xml
<!-- res/xml/locales_config.xml -->
<locale-config xmlns:android="http://schemas.android.com/apk/res/android">
    <locale android:name="ja" />
    <locale android:name="en" />
    <locale android:name="zh" />
</locale-config>
```

```xml
<!-- AndroidManifest.xml -->
<application
    android:localeConfig="@xml/locales_config">
```

---

## チェックリスト

- [ ] 全テキストを`strings.xml`に定義（ハードコード禁止）
- [ ] デフォルト言語のstrings.xmlが全キーを含む
- [ ] レイアウトがドイツ語（長い単語）で崩れないか確認
- [ ] 日付・数値はLocale対応フォーマッターを使用
- [ ] RTLサポートを有効化
- [ ] Google Playストアの説明文も多言語対応

---

## まとめ

- `values-XX/strings.xml`で言語別テキスト管理
- `stringResource()`でCompose内参照
- `pluralStringResource()`で複数形対応
- 日付・通貨は`Locale.getDefault()`で自動フォーマット
- Android 13+の`Per-App Language`で言語切り替え

---

8種類のAndroidアプリテンプレート（多言語対応可能な設計）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [アクセシビリティ対応入門](https://zenn.dev/myougatheaxo/articles/android-accessibility-basics-2026)
- [Google Play公開チェックリスト](https://zenn.dev/myougatheaxo/articles/google-play-publish-guide-2026)
- [Composeレイアウト入門](https://zenn.dev/myougatheaxo/articles/compose-layout-basics-2026)
