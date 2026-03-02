---
title: "MVVM設計の全体像 — AIが生成するAndroidアプリの構造を図解する"
emoji: "🏗️"
type: "tech"
topics: ["android", "kotlin", "mvvm", "architecture", "jetpackcompose"]
published: true
---

## この記事で学べること

AIでAndroidアプリを生成すると、複数のファイルが作られます。「どのファイルが何をしているのか」を理解するための記事です。

---

## 全体構造（レイヤー図）

```
┌─────────────────────────────┐
│        UI Layer             │
│   Composable (@Composable)  │
│   画面の見た目を定義        │
└──────────┬──────────────────┘
           │ collectAsState()
┌──────────▼──────────────────┐
│      ViewModel Layer        │
│   ViewModel + StateFlow     │
│   UIの状態管理・ロジック    │
└──────────┬──────────────────┘
           │ suspend fun
┌──────────▼──────────────────┐
│     Repository Layer        │
│   データ操作の抽象化        │
│   (省略されることもある)     │
└──────────┬──────────────────┘
           │ DAO
┌──────────▼──────────────────┐
│       Data Layer            │
│   Room Database + Entity    │
│   SQLiteへの永続化          │
└─────────────────────────────┘
```

---

## 各レイヤーの役割

### UI Layer（画面）

```kotlin
@Composable
fun HabitListScreen(viewModel: HabitViewModel) {
    val habits by viewModel.habits.collectAsState()

    LazyColumn {
        items(habits) { habit ->
            HabitItem(
                habit = habit,
                onToggle = { viewModel.toggleHabit(habit) },
                onDelete = { viewModel.deleteHabit(habit) }
            )
        }
    }
}
```

**やること**: データを表示する、ユーザー操作をViewModelに伝える
**やらないこと**: データベースアクセス、ビジネスロジック

### ViewModel Layer（状態管理）

```kotlin
class HabitViewModel(private val dao: HabitDao) : ViewModel() {

    val habits: StateFlow<List<Habit>> = dao.getAllHabits()
        .stateIn(viewModelScope, SharingStarted.Lazily, emptyList())

    fun toggleHabit(habit: Habit) {
        viewModelScope.launch {
            dao.update(habit.copy(completed = !habit.completed))
        }
    }

    fun deleteHabit(habit: Habit) {
        viewModelScope.launch {
            dao.delete(habit)
        }
    }
}
```

**やること**: UIの状態を保持する、ビジネスロジックを実行する
**やらないこと**: UIの描画、Contextの保持

### Data Layer（データ永続化）

```kotlin
@Entity(tableName = "habits")
data class Habit(
    @PrimaryKey(autoGenerate = true) val id: Int = 0,
    val name: String,
    val completed: Boolean = false,
    val createdAt: Long = System.currentTimeMillis()
)

@Dao
interface HabitDao {
    @Query("SELECT * FROM habits ORDER BY createdAt DESC")
    fun getAllHabits(): Flow<List<Habit>>

    @Insert
    suspend fun insert(habit: Habit)

    @Update
    suspend fun update(habit: Habit)

    @Delete
    suspend fun delete(habit: Habit)
}

@Database(entities = [Habit::class], version = 1)
abstract class AppDatabase : RoomDatabase() {
    abstract fun habitDao(): HabitDao
}
```

**やること**: SQLiteへのデータ保存/取得
**やらないこと**: UI表示、ビジネスロジック判断

---

## データの流れ

### 読み取り（リアクティブ）

```
Room DB → Flow<List<Habit>> → DAO → ViewModel → StateFlow → Composable
```

データベースの変更が**自動的にUIに反映**されます。手動でリフレッシュする必要なし。

### 書き込み

```
ユーザー操作 → Composable → ViewModel.addHabit() → DAO.insert() → Room DB
                                                                      ↓
                                                          Flow が自動で新データを通知
                                                                      ↓
                                                          UI が自動で再描画
```

---

## ファイル構成

AIが生成する典型的なファイル構成：

```
app/src/main/java/com/example/habittracker/
├── data/
│   ├── Habit.kt          (Entity)
│   ├── HabitDao.kt       (DAO)
│   └── AppDatabase.kt    (Database)
├── ui/
│   ├── theme/
│   │   ├── Color.kt
│   │   ├── Theme.kt
│   │   └── Type.kt
│   ├── HabitListScreen.kt
│   ├── AddHabitScreen.kt
│   └── HabitItem.kt
├── viewmodel/
│   ├── HabitViewModel.kt
│   └── HabitViewModelFactory.kt
└── MainActivity.kt
```

---

## なぜMVVMなのか

| メリット | 説明 |
|---------|------|
| テスト容易 | ViewModelを単独でテストできる |
| 関心の分離 | UI変更がデータ層に影響しない |
| 画面回転対応 | ViewModelがデータを保持 |
| リアクティブ | データ変更が自動でUIに反映 |

**デメリット**: 小さいアプリには過剰設計に見える。でもAndroidの公式推奨パターンなので、AIも全てこの構造で生成します。

---

## 新機能を追加するとき

「ソート機能を追加したい」場合の変更箇所：

1. **DAO**: `@Query("SELECT * FROM habits ORDER BY name ASC")`を追加
2. **ViewModel**: ソートの切り替えロジックを追加
3. **UI**: ソートボタンを追加

各レイヤーの責任が明確なので、**何を変更すればいいかが一目瞭然**です。

---

## まとめ

- MVVM = Model（データ）+ View（UI）+ ViewModel（状態管理）
- データは`Flow`で自動的にUIに流れる
- AIはこの構造を正しく自動生成する
- 新機能追加は各レイヤーの役割に沿って変更するだけ

---

8種類のAndroidアプリテンプレート（全てMVVM + Room + Compose設計）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [AIでAndroidアプリを47秒で作った話](https://zenn.dev/myougatheaxo/articles/ai-android-app-47seconds)
- [ViewModel + StateFlow入門](https://zenn.dev/myougatheaxo/articles/android-viewmodel-stateflow-2026)
- [Room Databaseでプライバシー設計入門](https://zenn.dev/myougatheaxo/articles/room-database-privacy-2026)
- [DI（依存性注入）の選び方](https://zenn.dev/myougatheaxo/articles/android-di-manual-vs-hilt-2026)
