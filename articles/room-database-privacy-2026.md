---
title: "Room Databaseで「データが端末から出ない」アプリを作る【プライバシー設計入門】"
emoji: "🔒"
type: "tech"
topics: ["Android", "Room", "Kotlin", "プライバシー", "セキュリティ"]
published: true
---

## はじめに

「このアプリ、データをどこに送ってるんだろう？」

ユーザーの94%は気にしていない。でも残りの6%は気にしている。そしてその6%が、レビューを書き、SNSでシェアし、有料アプリを買う層です。

この記事では、**Room Databaseを使ってデータが端末から一切出ないアプリを作る方法**を解説します。

---

## Room Databaseとは

GoogleがAndroid向けに提供する**SQLite抽象化ライブラリ**です。

SQLiteを直接使うよりも：
- コンパイル時にSQLの文法チェックが入る
- Kotlinのdata classと自動マッピング
- FlowやCoroutineとの連携が標準サポート

---

## 3つの構成要素

### 1. Entity（データの定義）

```kotlin
@Entity(tableName = "habits")
data class Habit(
    @PrimaryKey(autoGenerate = true) val id: Int = 0,
    val name: String,
    val createdAt: Long = System.currentTimeMillis(),
    val streak: Int = 0
)
```

`data class` + `@Entity` だけで、SQLiteのテーブル定義が完了します。

### 2. DAO（データアクセス）

```kotlin
@Dao
interface HabitDao {
    @Query("SELECT * FROM habits ORDER BY createdAt DESC")
    fun getAllHabits(): Flow<List<Habit>>

    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insertHabit(habit: Habit)

    @Delete
    suspend fun deleteHabit(habit: Habit)
}
```

**ポイント：**

- `Flow<List<Habit>>` — データが変更されるとUIが自動更新される
- `suspend fun` — メインスレッドをブロックしない非同期処理

### 3. Database（データベース本体）

```kotlin
@Database(entities = [Habit::class], version = 1)
abstract class AppDatabase : RoomDatabase() {
    abstract fun habitDao(): HabitDao

    companion object {
        @Volatile private var INSTANCE: AppDatabase? = null

        fun getDatabase(context: Context): AppDatabase {
            return INSTANCE ?: synchronized(this) {
                Room.databaseBuilder(
                    context.applicationContext,
                    AppDatabase::class.java,
                    "app_database"
                ).build().also { INSTANCE = it }
            }
        }
    }
}
```

シングルトンパターンで、アプリ全体で1つのインスタンスを共有。

---

## なぜプライバシー設計なのか

Room Databaseを使うと、データの保存先は**端末内のSQLiteファイル**のみ。

これが意味すること：

- **INTERNET権限が不要** — データを外部に送る経路がない
- **サーバーが不要** — 運用コストゼロ
- **GDPR/個人情報保護法に抵触しない** — 個人データを収集していないので
- **オフラインで動作** — 通信環境に依存しない

### AndroidManifest.xml

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.example.habittracker">
    <!-- 権限の宣言なし -->
    <application ...>
```

`<uses-permission>` タグが一つもない。これが**最強のプライバシーステートメント**です。

---

## 無料アプリとの比較

| 項目 | 無料アプリ（一般的） | Room使用アプリ |
|------|---------------------|---------------|
| INTERNET | 必須 | 不要 |
| データ送信先 | 広告SDK + 分析SDK | なし |
| トラッカー数 | 平均5.3個 | 0個 |
| オフライン動作 | 一部機能のみ | 完全動作 |
| プライバシーポリシー | 長い | 「データ収集なし」で終わり |

---

## AIが生成するRoomコードの品質

Claude Codeに「ハビットトラッカーを作って」と指示すると、上記のRoom構成が**47秒で自動生成**されます。

AIが自動的に選択するパターン：
- `@PrimaryKey(autoGenerate = true)` — ID衝突を防止
- `Flow` 返却 — リアクティブUI
- `suspend` 関数 — 非同期処理
- `OnConflictStrategy.REPLACE` — 明示的な競合解決
- シングルトンDatabase — メモリ効率

ベテランエンジニアのレビュー結果：**「正しい」**。

---

## まとめ

- Room = SQLiteの安全な抽象化
- 3つの構成要素（Entity, DAO, Database）だけ覚えれば使える
- INTERNET権限なし = 最強のプライバシー設計
- AIが生成するRoomコードはプロダクション品質
- 「データが端末から出ない」は2026年のアプリの差別化ポイント

---

:::message
8種類のAndroidアプリテンプレート（全てRoom Database使用、INTERNET権限なし）を[Gumroad](https://myougatheax.gumroad.com)で公開中。ソースコード100%確認可能。
:::

---

## 関連記事

- [Claude Codeに「Androidアプリ作って」と言ったら47秒で完成した](https://zenn.dev/myougatheaxo/articles/ai-android-app-47seconds)
- [AIが生成したコードをベテランエンジニアにレビューしてもらった](https://zenn.dev/myougatheaxo/articles/ai-code-quality-review)
- [「無料アプリ」の本当のコスト](https://zenn.dev/myougatheaxo/articles/free-apps-hidden-cost-2026)
- [AIアプリ開発で月3万円の副収入ロードマップ](https://zenn.dev/myougatheaxo/articles/ai-side-income-roadmap-2026)
- [2026年版 AIコーディングツール完全ガイド](https://zenn.dev/myougatheaxo/articles/ai-coding-tools-complete-2026)
