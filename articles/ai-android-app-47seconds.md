---
title: "Claude Codeに「Androidアプリ作って」と言ったら47秒で完成した"
emoji: "🦎"
type: "tech"
topics: ["ClaudeCode", "Android", "Kotlin", "AI", "個人開発"]
published: true
---

## はじめに

Claude Codeに **「習慣トラッカーアプリを作って」** と一言だけ伝えた結果、**47秒で動くAndroidアプリが完成**しました。

この記事では、AIが生成したコードの品質を検証し、「実用レベルなのか」を分析します。

## 生成されたコードの構成

```
- 言語: Kotlin
- UI: Jetpack Compose + Material3
- データ保存: Room Database（SQLite）
- アーキテクチャ: MVVM + Repository pattern
- テーマ: ダークモード対応
```

## 品質チェック 3つのポイント

### 1. エラーハンドリング

```kotlin
try {
    habitDao.insert(habit)
} catch (e: SQLiteConstraintException) {
    emit(Result.Error("Habit already exists"))
}
```

例外処理がきちんとある。エッジケースも考慮。

### 2. データ構造

```kotlin
@Entity(tableName = "habits")
data class Habit(
    @PrimaryKey(autoGenerate = true) val id: Int = 0,
    val name: String,
    val createdAt: Long = System.currentTimeMillis(),
    val streak: Int = 0
)
```

Room Entityとして適切に定義。autoGenerate、デフォルト値、型安全。

### 3. 関心の分離

- `HabitDao` - データアクセス層
- `HabitRepository` - ビジネスロジック
- `HabitViewModel` - UI状態管理
- `HabitScreen` - 表示コンポーネント

レイヤーがきちんと分離されている。初心者が書くとDAO直呼びしがちな部分。

## ベテランエンジニアの反応

90年代からコードを書いてる叔父にAI生成コードを見せた結果：

> 「正しい」

「正しい」の意味：
- エラーハンドリングがある（ユーザーが間違った操作をした時の処理）
- データ構造が整理されている（情報がランダムに保存されていない）
- 各部分が一つのことだけを担当している（責務の分離）

## AIが最初にやったのはコードを書くことではない

意外だったのは、AIがコードを書く前に**質問してきた**こと：

- 「誰が使うアプリ？」
- 「10秒以内にやりたいことは何？」
- 「間違った操作をしたらどうする？」
- 「オフラインで動く必要がある？」
- 「セッション間でデータを保持する？」

これは優秀な開発者が必ず聞く質問です。AIはコーディングだけでなく、**設計思考もできる**。

## 11分で「自分のアプリ」にする手順

1. テンプレートをダウンロード（30秒）
2. `strings.xml` のアプリ名を変更（2分）
3. `Theme.kt` のカラーを変更（90秒）
4. Android Studioで `./gradlew assembleDebug`（5分）
5. USBでスマホにインストール（2分）

コーディング経験ゼロでも、**設定ファイルの値を書き換えるだけ**。

## まとめ

- AI生成コードは「おもちゃ」レベルではない
- 適切なプロンプトなら、プロと同等のアーキテクチャ
- 「何を作るか」は人間、「どう作るか」はAI
- **47秒で完成する時代に、ゼロから書く理由はない**

---

:::message
8種類のAndroidアプリテンプレート（Kotlin + Compose + Material3）を[Gumroad](https://myougatheax.gumroad.com)で公開中。プロンプト設計済み、ビルド手順付き。
:::
