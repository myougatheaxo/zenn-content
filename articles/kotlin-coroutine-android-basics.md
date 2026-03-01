---
title: "Kotlin Coroutineがわからない人へ — AIが書くコードで理解するAndroid非同期処理"
emoji: "🔄"
type: "tech"
topics: ["Kotlin", "Android", "Coroutine", "JetpackCompose", "初心者"]
published: true
---

## Coroutineって何？

一言で：**メインスレッドを止めずにデータベースの読み書きをする仕組み**です。

Androidでは、メインスレッド（UIスレッド）が止まると画面がフリーズします。データベースの読み書きは時間がかかるので、メインスレッドで実行すると**ANR（Application Not Responding）**エラーが出ます。

Coroutineを使うと、データベース操作を**別のスレッドで実行しつつ、結果をメインスレッドに返す**ことができます。

---

## AIが生成するCoroutineコード

Claude Codeに「ハビットトラッカーを作って」と指示すると、以下のコードが自動生成されます。

### DAO（データアクセス）

```kotlin
@Dao
interface HabitDao {
    @Query("SELECT * FROM habits ORDER BY createdAt DESC")
    fun getAllHabits(): Flow<List<Habit>>

    @Insert
    suspend fun insertHabit(habit: Habit)

    @Delete
    suspend fun deleteHabit(habit: Habit)
}
```

ここに2つのCoroutine要素があります：

1. **`suspend`** — この関数は一時停止できる（別スレッドで実行される）
2. **`Flow`** — データの変更をリアルタイムで通知するストリーム

### ViewModel

```kotlin
class HabitViewModel(application: Application) : AndroidViewModel(application) {
    private val repository: HabitRepository
    val allHabits: StateFlow<List<Habit>>

    init {
        val dao = AppDatabase.getDatabase(application).habitDao()
        repository = HabitRepository(dao)
        allHabits = repository.allHabits
            .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), emptyList())
    }

    fun addHabit(name: String) = viewModelScope.launch {
        repository.insert(Habit(name = name))
    }
}
```

ここにも2つのCoroutine要素があります：

3. **`viewModelScope.launch`** — ViewModelのライフサイクルに紐づいたCoroutine起動
4. **`stateIn`** — FlowをStateFlowに変換（Composeで使うため）

---

## 4つのキーワードだけ覚える

| キーワード | 意味 | 使いどころ |
|-----------|------|-----------|
| `suspend` | この関数は別スレッドで実行される | DAO、Repository |
| `Flow` | データの変更をリアルタイムで通知 | DAOの返り値 |
| `viewModelScope.launch` | Coroutineを起動する | ViewModel |
| `stateIn` | FlowをStateFlowに変換 | ViewModel |

これだけです。他の概念（`Dispatchers`, `withContext`, `async/await`）は、シンプルなアプリでは不要です。

---

## なぜAIはCoroutineを正しく書けるのか

AIが生成するCoroutineコードが正しい理由：

1. **パターンが確立されている** — Room + Flow + ViewModelの組み合わせはGoogleの公式ドキュメントに明記されている
2. **学習データが豊富** — 何万ものAndroidプロジェクトから学習している
3. **バリエーションが少ない** — 正しい書き方がほぼ1つしかない

ベテランエンジニアのレビュー結果でも、Coroutineの使い方は**「正しい」**と評価されています。

---

## よくある間違い（AIは犯さないが人間は犯す）

### 間違い1: suspendをつけ忘れる

```kotlin
// ✗ クラッシュする
@Insert
fun insertHabit(habit: Habit)

// ○ 正しい
@Insert
suspend fun insertHabit(habit: Habit)
```

### 間違い2: メインスレッドでDB操作

```kotlin
// ✗ ANRエラー
fun addHabit(name: String) {
    repository.insert(Habit(name = name)) // メインスレッドで実行してしまう
}

// ○ 正しい
fun addHabit(name: String) = viewModelScope.launch {
    repository.insert(Habit(name = name)) // 別スレッドで実行
}
```

### 間違い3: FlowではなくListを返す

```kotlin
// ✗ UIが自動更新されない
@Query("SELECT * FROM habits")
fun getAllHabits(): List<Habit>

// ○ UIが自動更新される
@Query("SELECT * FROM habits")
fun getAllHabits(): Flow<List<Habit>>
```

---

## まとめ

- Coroutineは「メインスレッドを止めない」ための仕組み
- `suspend`, `Flow`, `viewModelScope.launch`, `stateIn` の4つだけ覚える
- AIが生成するCoroutineコードはパターンが確立されており正確
- 人間が書くと間違えやすいポイントをAIは自動的に回避する

---

:::message
AIが生成したCoroutine + Room + Composeのコードを実際に動かしてみたい方へ。8種類のAndroidアプリテンプレートを[Gumroad](https://myougatheax.gumroad.com)で公開中。
:::

---

## 関連記事

- [Claude Codeに「Androidアプリ作って」と言ったら47秒で完成した](https://zenn.dev/myougatheaxo/articles/ai-android-app-47seconds)
- [AIが生成したコードをベテランエンジニアにレビューしてもらった](https://zenn.dev/myougatheaxo/articles/ai-code-quality-review)
- [Room Databaseで「データが端末から出ない」アプリを作る](https://zenn.dev/myougatheaxo/articles/room-database-privacy-2026)
- [Jetpack Composeで最初に覚えるべき5つのコンポーネント](https://zenn.dev/myougatheaxo/articles/claude-md-best-practices-2026)

:::message
Zenn Booksでも詳しく解説: [AIでAndroidアプリを作る実践ガイド](https://zenn.dev/myougatheaxo/books/ai-android-app-builder) (¥980)
:::
