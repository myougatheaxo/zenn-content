---
title: "カスタマイズガイド — 色・名前・機能を変える"
---

ハビットトラッカーが動くようになりました。でもこのまま「ハビットトラッカー」として売るのは難しいです。似たようなアプリがすでに山ほどあるから。

この章では、**同じアーキテクチャを使いながら「別のアプリ」に変える方法**を学びます。

色を変える、アプリ名を変える、機能を追加する、テーマを変える。これだけで「全然違うアプリ」に見えます。

---

## カスタマイズ1: アプリ名とパッケージ名を変える

### アプリ名

`app/src/main/res/values/strings.xml` を開きます：

```xml
<resources>
    <string name="app_name">習慣トラッカー</string>
</resources>
```

ここを好きな名前に変えるだけです：

```xml
<string name="app_name">みょうが習慣メモ</string>
```

### パッケージ名（重要）

パッケージ名はGoogle Playでアプリを一意に識別するIDです。同じパッケージ名のアプリは1つしか存在できません。

`app/build.gradle.kts` の `applicationId` を変えます：

```kotlin
defaultConfig {
    applicationId = "com.yourname.habittracker"  // ← これを変える
    // ...
}
```

おすすめの命名規則：`com.あなたの名前.アプリ名`

例：
- `com.myouga.habitpro`
- `com.tanaka.watertracker`
- `io.github.yourname.moodlog`

パッケージ名を変えたあとは、全ファイルのpackage宣言も変更が必要です。AIへの指示：

```
パッケージ名を com.yourname.habittracker から
com.myouga.habitpro に変更して。
全ファイルのpackage宣言とimport文を更新して。
```

---

## カスタマイズ2: カラーテーマを変える

Jetpack Composeのテーマは `app/src/main/java/.../ui/theme/` に入っています。

`Color.kt` でカラーパレットを定義：

```kotlin
// ui/theme/Color.kt

package com.yourname.habittracker.ui.theme

import androidx.compose.ui.graphics.Color

// ライトテーマ用
val PrimaryGreen = Color(0xFF2E7D32)
val SecondaryTeal = Color(0xFF00695C)
val BackgroundLight = Color(0xFFF1F8E9)

// ダークテーマ用
val PrimaryGreenDark = Color(0xFF81C784)
val SecondaryTealDark = Color(0xFF4DB6AC)
val BackgroundDark = Color(0xFF1B2D1C)
```

`Theme.kt` でMaterial3のカラースキームに割り当て：

```kotlin
// ui/theme/Theme.kt

private val LightColorScheme = lightColorScheme(
    primary = PrimaryGreen,
    secondary = SecondaryTeal,
    background = BackgroundLight,
    surface = Color.White,
    onPrimary = Color.White,
    onSecondary = Color.White,
    onBackground = Color(0xFF1C1B1F),
    onSurface = Color(0xFF1C1B1F),
)
```

**Material Theme Builder を使う（おすすめ）：**

https://m3.material.io/theme-builder にアクセスして、好みの色を選ぶと `Theme.kt` と `Color.kt` のコードを自動生成してくれます。そのままコピーするだけでオシャレなテーマが完成します。

AIへの指示：

```
テーマカラーをピンク系に変えて。
ライトテーマ: primary=#E91E63, secondary=#F06292
ダークテーマ: primary=#F48FB1, secondary=#FCE4EC
Color.kt と Theme.kt を更新して。
```

---

## カスタマイズ3: フォントを変える

Google Fontsのフォントを使う例（Noto Sans JPで日本語表示を改善）：

`build.gradle.kts` に追加：

```kotlin
implementation("androidx.compose.ui:ui-text-google-fonts:1.7.3")
```

`ui/theme/Type.kt` を作成：

```kotlin
package com.yourname.habittracker.ui.theme

import androidx.compose.material3.Typography
import androidx.compose.ui.text.TextStyle
import androidx.compose.ui.text.font.FontFamily
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.text.googlefonts.Font
import androidx.compose.ui.text.googlefonts.GoogleFont
import androidx.compose.ui.unit.sp
import com.yourname.habittracker.R

val provider = GoogleFont.Provider(
    providerAuthority = "com.google.android.gms.fonts",
    providerPackage = "com.google.android.gms",
    certificates = R.array.com_google_android_gms_fonts_certs
)

val NotoSansJP = FontFamily(
    Font(googleFont = GoogleFont("Noto Sans JP"), fontProvider = provider)
)

val AppTypography = Typography(
    bodyLarge = TextStyle(
        fontFamily = NotoSansJP,
        fontWeight = FontWeight.Normal,
        fontSize = 16.sp
    ),
    titleMedium = TextStyle(
        fontFamily = NotoSansJP,
        fontWeight = FontWeight.SemiBold,
        fontSize = 16.sp
    )
)
```

---

## カスタマイズ4: データモデルを拡張する

ハビットトラッカーを「水分記録アプリ」に変えるとしたら：

AIへの指示：

```
ハビットトラッカーを水分摂取トラッカーに変えて。

変更点:
- Habitエンティティ → WaterEntry (date: String, amountMl: Int)
- 毎日の目標: 2000ml
- 1日の記録を積み上げられる（例: 250ml × 8回）
- 今日の合計と達成率を表示
- 7日間の履歴グラフ（BarChartをComposeで簡易実装）

パッケージ名: com.yourname.watertracker
アプリ名: 水分記録
```

これだけで、同じアーキテクチャを使った全然違うアプリが作れます。

---

## カスタマイズ5: 機能を追加する

**通知機能を追加する**（習慣リマインダー）：

AIへの指示：

```
毎日指定した時刻にリマインダー通知を送る機能を追加して。

要件:
- ユーザーが通知時刻を設定できる（TimePickerダイアログ）
- AlarmManagerで毎日繰り返す
- 通知をタップするとアプリが開く
- Android 13+のPOST_NOTIFICATIONS権限リクエスト対応
```

生成されるコードの要点（確認ポイント）：

```kotlin
// AndroidManifest.xmlに追加が必要
<uses-permission android:name="android.permission.POST_NOTIFICATIONS"/>
<uses-permission android:name="android.permission.SCHEDULE_EXACT_ALARM"/>

// AlarmManagerの設定
val alarmManager = getSystemService(Context.ALARM_SERVICE) as AlarmManager
val pendingIntent = PendingIntent.getBroadcast(
    context, habitId, intent,
    PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_IMMUTABLE
)
alarmManager.setRepeating(
    AlarmManager.RTC_WAKEUP,
    triggerAtMillis,
    AlarmManager.INTERVAL_DAY,
    pendingIntent
)
```

**ウィジェット機能を追加する**：

```
ホーム画面ウィジェットを追加して。
サイズ: 4x2
内容: 今日の習慣リストと達成状況（チェックボックス付き）
Glance APIを使って実装して。
```

---

## カスタマイズ6: 別のアプリテンプレートに応用する

同じ「MVVM + Room + Compose」のパターンで作れるアプリ一覧：

| アプリ | エンティティ | 特徴 |
|--------|-------------|------|
| 家計簿 | Expense(date, amount, category, memo) | カテゴリ別グラフ |
| 体重記録 | WeightEntry(date, weightKg) | 折れ線グラフ |
| 読書記録 | Book(title, author, startDate, finishDate, rating, memo) | 進捗管理 |
| 気分日記 | MoodEntry(date, mood: Int, memo) | 気分スコアのヒートマップ |
| タスク管理 | Task(title, deadline, priority, isDone) | 締切順ソート |
| 学習記録 | StudySession(subject, durationMin, date) | 週次サマリー |

どれも「AIにエンティティを説明して、前章のハビットトラッカーのアーキテクチャを参考に作って」という指示で8割は作れます。

---

## カスタマイズの黄金ルール

1. **一度に変えすぎない**: 1つ変えてビルド・確認 → 次の変更。まとめて変えるとバグの原因が特定できなくなります。

2. **変更前にバックアップ**: Gitを使うか、ファイルをコピーしておく。`git init && git add . && git commit -m "initial"` で保存してから変更開始。

3. **AIにコンテキストを渡す**: 「さっき作ったハビットトラッカーのコードに〇〇を追加して」より「このコード全体（貼り付け）に対して〇〇を追加して」の方が精度が上がります。

4. **エラーはそのままAIに渡す**: エラーメッセージをコピーして「これを直して」と言うだけ。自分でエラーを解読しなくていいです。

---

## まとめ

- アプリ名・パッケージ名の変更はリリース前に必ず実施
- カラーテーマはMaterial Theme Builderで自動生成が楽
- データモデルを変えれば全く別のアプリになる
- 機能追加もAIへの指示で対応可能（通知・ウィジェットなど）
- 同じアーキテクチャで複数アプリを作ると効率が上がる

次の章では、完成したアプリをGoogle Playに出す手順を説明します。
