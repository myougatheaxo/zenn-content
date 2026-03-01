---
title: "AIが生成したAndroidコードをベテランエンジニアに見せたら「正しい」と言われた"
emoji: "🔍"
type: "tech"
topics: ["Android", "Kotlin", "AI", "コードレビュー", "JetpackCompose"]
published: true
---

## 実験の概要

Claude Codeに「ハビットトラッカーを作って」と指示した。47秒後、Kotlin + Jetpack Compose + Room + MVVMのプロジェクトが生成された。

このコードを、90年代からコードを書いているベテランエンジニアに見せた。

結果：**「正しい」**。

この記事では、具体的にどの部分が「正しい」のか、そしてどこに限界があるのかを技術的に検証します。

## 生成されたコードの構造

```
app/src/main/java/com/example/habittracker/
├── data/
│   ├── Habit.kt          ← Entity
│   ├── HabitDao.kt       ← DAO
│   └── AppDatabase.kt    ← RoomDatabase
├── repository/
│   └── HabitRepository.kt ← Repository
├── viewmodel/
│   └── HabitViewModel.kt  ← ViewModel
└── ui/
    ├── HabitScreen.kt     ← Composable UI
    └── theme/
        └── Theme.kt       ← Material3テーマ
```

MVVM + Repository パターン。Googleが推奨するAndroidアーキテクチャそのもの。

## 評価ポイント1: データ層

```kotlin
@Entity(tableName = "habits")
data class Habit(
    @PrimaryKey(autoGenerate = true) val id: Int = 0,
    val name: String,
    val createdAt: Long = System.currentTimeMillis(),
    val streak: Int = 0
)
```

**正しい点：**
- `@Entity` + `@PrimaryKey(autoGenerate = true)` — Room Entityの標準的な書き方
- `data class` — イミュータブル。値の比較が`equals()`で正しく動く
- デフォルト値あり — `id = 0`, `streak = 0` でインスタンス生成が安全

**ジュニア開発者がよくやるミス：**
- `var`を使ってミュータブルにする → 予期しない変更が起きる
- `autoGenerate`を忘れてIDを手動管理 → 衝突の原因
- `createdAt`をStringにする → ソートや比較ができない

## 評価ポイント2: DAO

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

**正しい点：**
- `Flow<List<Habit>>` — リアクティブなデータストリーム。UIが自動更新される
- `suspend fun` — Coroutineで非同期実行。メインスレッドをブロックしない
- `OnConflictStrategy.REPLACE` — 重複時の明示的な戦略

**ジュニアがやるミス：**
- `List<Habit>`を返してUIを手動更新 → 同期漏れが起きる
- `suspend`をつけずにメインスレッドで実行 → ANRエラー

## 評価ポイント3: Repository

```kotlin
class HabitRepository(private val habitDao: HabitDao) {
    val allHabits: Flow<List<Habit>> = habitDao.getAllHabits()

    suspend fun insert(habit: Habit) = habitDao.insertHabit(habit)
    suspend fun delete(habit: Habit) = habitDao.deleteHabit(habit)
}
```

**正しい点：**
- DAOをラップして抽象化 — データソースの切り替えが容易
- `Flow`をそのまま公開 — ViewModelでcollectするだけ

「Repositoryって必要？DAOを直接呼べばいいのでは？」という疑問もある。小規模アプリでは確かにオーバーかもしれない。だが、後でAPIデータソースを追加する場合、Repositoryがあれば変更箇所が1つで済む。

## 評価ポイント4: ViewModel

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

    fun deleteHabit(habit: Habit) = viewModelScope.launch {
        repository.delete(habit)
    }
}
```

**正しい点：**
- `StateFlow` + `stateIn` — Composeとの連携に最適
- `WhileSubscribed(5000)` — 画面離脱後5秒で購読停止（リソース効率）
- `viewModelScope.launch` — ViewModelのライフサイクルに紐づいたCoroutine

## 評価ポイント5: 権限

```xml
<!-- AndroidManifest.xml -->
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.example.habittracker">
    <!-- 権限なし -->
    <application ...>
```

**INTERNET権限すらない**。全データはRoomデータベース（端末内SQLite）に保存される。

比較：

| 権限 | 無料アプリ | AI生成アプリ |
|------|-----------|------------|
| INTERNET | ✅ | ❌ |
| ACCESS_FINE_LOCATION | ✅ | ❌ |
| READ_CONTACTS | ✅ | ❌ |
| AD_ID | ✅ | ❌ |
| バックグラウンド実行 | ✅ | ❌ |

## AIコードの限界

正直に書く。

### 生成されないもの

- **ユニットテスト** — テストコードは生成されなかった
- **CI/CDパイプライン** — GitHub ActionsやBitrise設定なし
- **ProGuardルール** — リリースビルドの難読化設定なし
- **アクセシビリティ対応** — contentDescriptionの設定が不完全

### 改善の余地

- DIフレームワーク（Hilt/Koin）を使っていない — 手動DI
- Navigation Composeを使っていない — 単画面前提
- エッジケースのテスト不足

## 結論

| 項目 | 評価 |
|------|------|
| アーキテクチャ | ◎ MVVM + Repository（Google推奨） |
| データ層 | ◎ Room + Flow + Coroutine |
| UI層 | ○ Compose + Material3 |
| エラーハンドリング | ○ 基本的なものは存在 |
| テスト | △ なし |
| セキュリティ | ◎ 不要な権限ゼロ |

シンプルなアプリにおいて、AIが生成するコードは**プロダクション品質**に達している。ただし、複雑なアプリ（マルチ画面、API連携、認証）では人間によるレビューと補完が必要。

---

:::message
8種類のAndroidアプリテンプレート（Kotlin + Compose + Material3）を[Gumroad](https://myougatheax.gumroad.com)で公開中。広告なし、トラッキングなし、ソースコード100%確認可能。
:::

---

## 関連記事

- [Claude Codeに「Androidアプリ作って」と言ったら47秒で完成した](https://zenn.dev/myougatheaxo/articles/ai-android-app-47seconds)
- [コーディング経験ゼロからAndroidアプリを11分で動かす](https://zenn.dev/myougatheaxo/articles/android-app-no-code-guide)
- [Claude Code vs Cursor vs GitHub Copilot 比較](https://zenn.dev/myougatheaxo/articles/claude-code-vs-cursor-copilot)
- [「無料アプリ」の本当のコスト](https://zenn.dev/myougatheaxo/articles/free-apps-hidden-cost-2026)

:::message
8種類のAndroidアプリテンプレートを[Gumroad](https://myougatheax.gumroad.com)で公開中。
:::
