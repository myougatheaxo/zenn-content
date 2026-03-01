---
title: "生成されたコードを理解する — MVVM・Room・Compose"
---

前の章で動くアプリが完成しました。でも、「コードの意味はよくわからない...」という状態のままGoogle Playに出すのは少し不安ですよね。

この章では、**「分かっているふりをしなくていい」レベルの理解**を目指します。

エンジニアが10年かけて習得するような深い知識は不要です。「なぜこの構造になっているのか」「何かトラブルが起きたときにどこを見ればいいか」が分かれば十分です。

---

## なぜ「アーキテクチャ」が必要なのか

Androidアプリは複雑になりがちです。画面が増え、データが増え、機能が増えると、コードが「スパゲッティ」になります。

アーキテクチャとは、**コードをどこに書くかのルール**です。

ハビットトラッカーで使っているMVVMは、責任を3層に分けます：

```
UI（Compose画面）
    ↕ データを受け取る / ユーザー操作を伝える
ViewModel（ビジネスロジック）
    ↕ データの取得・加工を依頼する
Repository + Room（データ層）
    ↕ 実際にDBからデータを読み書きする
```

この分離がなぜ重要か：

- **UIはViewModelを知っている**が、ViewModelはUIを知らない → 画面が変わってもロジックを書き直さなくていい
- **ViewModelはRepositoryを知っている**が、RepositoryはViewModelを知らない → データソースを変えてもロジックに影響しない
- それぞれの部分を**独立してテスト**できる

---

## MVVM の各層を見る

### M（Model）= データの定義

`Habit.kt` がモデルです：

```kotlin
@Entity(tableName = "habits")
data class Habit(
    @PrimaryKey(autoGenerate = true)
    val id: Int = 0,
    val name: String,
    val createdAt: Long = System.currentTimeMillis(),
    val completedDates: String = ""
)
```

`@Entity` はRoomに「これをデータベースのテーブルとして扱って」と伝えるアノテーションです。`@PrimaryKey(autoGenerate = true)` は「idを自動で採番して」という意味。

`data class` はKotlinの特別なクラスで、`equals()`・`hashCode()`・`copy()` などが自動生成されます。データを保持するだけのクラスに使います。

### V（View）= 画面

`HabitScreen.kt` がビューです。`@Composable` がついた関数が画面を描画します：

```kotlin
@Composable
fun HabitScreen(viewModel: HabitViewModel = viewModel()) {
    val habits by viewModel.habits.collectAsStateWithLifecycle()
    // ...
}
```

`collectAsStateWithLifecycle()` は「ViewModelからのデータをリアルタイムで受け取って、変化があったら画面を再描画して」という意味です。これがComposeの**リアクティブUI**の核心です。

### VM（ViewModel）= 仲介役

`HabitViewModel.kt` がViewModelです：

```kotlin
class HabitViewModel(application: Application) : AndroidViewModel(application) {
    val habits: StateFlow<List<Habit>>

    fun addHabit(name: String) {
        viewModelScope.launch {
            repository.addHabit(name)
        }
    }
}
```

`StateFlow` は「データが変わったら通知してくれるリスト」です。`viewModelScope.launch` はバックグラウンドスレッドで処理を実行するためのコルーチンスコープです（UIを止めないために必要）。

---

## Room を理解する

RoomはAndroidの**SQLiteラッパー**です。直接SQLiteを書くよりずっと安全で、書き間違いをコンパイル時に検出してくれます。

### DAO（Data Access Object）

DAOはデータベース操作のインターフェースです：

```kotlin
@Dao
interface HabitDao {
    @Query("SELECT * FROM habits ORDER BY createdAt DESC")
    fun getAllHabits(): Flow<List<Habit>>

    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insertHabit(habit: Habit)
}
```

- `@Query` : SQL文を直接書く。引数の型が合わないとコンパイルエラーになる（安全）
- `@Insert` : SQLを書かなくてもインサート処理が使える
- `Flow<List<Habit>>` : データが変わるたびに自動で最新リストが流れてくる

### なぜ `suspend` がついているのか

```kotlin
suspend fun insertHabit(habit: Habit)
```

`suspend` は「この処理は時間がかかるので、完了するまで待って（でもUIスレッドは止めない）」という意味です。データベースへの書き込みはディスクアクセスを伴うため、必ずバックグラウンドで実行する必要があります。

`suspend` 関数は `coroutineScope` か `viewModelScope.launch { }` の中でしか呼べません。これがViewModelで `viewModelScope.launch` を使っている理由です。

---

## Jetpack Compose を理解する

ComposeはAndroidの新しいUI作成方法です。昔のXMLレイアウトより直感的です。

### Composable 関数

```kotlin
@Composable
fun HabitCard(habit: Habit, onToggle: () -> Unit) {
    Card {
        Row {
            Text(habit.name)
            IconButton(onClick = onToggle) {
                Icon(Icons.Default.Check, contentDescription = "チェック")
            }
        }
    }
}
```

`@Composable` がついた関数は「UI部品」です。関数の中に別の`@Composable`を入れることで、レゴブロックのように組み合わせます。

`Row` は横並び、`Column` は縦並び、`Box` は重ね合わせです。これさえ覚えれば、ほとんどのレイアウトが作れます。

### 状態（State）

Composeでは「状態が変わったら画面が自動で更新される」仕組みです：

```kotlin
var showAddDialog by remember { mutableStateOf(false) }
```

- `remember` : 再コンポーズ（再描画）をまたいで値を保持する
- `mutableStateOf` : 値が変わったことをComposeに通知できる
- `by` : `showAddDialog.value` と書かなくていいようにする委譲プロパティ

ViewModelからのデータは：

```kotlin
val habits by viewModel.habits.collectAsStateWithLifecycle()
```

`StateFlow` → `collectAsStateWithLifecycle()` → Composeの`State` に変換されます。

---

## AIにコードを説明させる

この章の内容がピンとこなくても大丈夫です。AIに聞けばいいです。

**便利なプロンプトパターン：**

```
このコードのXXXの部分が何をしているか、
初心者向けに100文字以内で説明して：

[コードをペースト]
```

```
このエラーが出た。原因と修正方法を教えて：

[エラーメッセージをペースト]
```

```
このコードをKotlin初心者でも理解できるように
コメントを追加して：

[コードをペースト]
```

コードを全部理解しなくても、「このファイルはUIの担当」「このファイルはデータの担当」という**大枠**が分かれば、どこを修正すればいいかが判断できます。

---

## よくある疑問 Q&A

**Q: `Flow` と `StateFlow` の違いは？**

A: `Flow` はコールドストリーム（誰かが収集を始めてから流れ出す）、`StateFlow` はホットストリーム（常に最新の値を持っていて、誰でもすぐ受け取れる）。ViewModelでは `StateFlow` を使うのが一般的です。

**Q: `suspend` 関数はどこで呼ぶの？**

A: `coroutineScope` か `viewModelScope.launch { }` の中です。ViewModelで `launch` を使えば、ViewModelが破棄されたとき自動的にキャンセルされます。

**Q: `@Entity` と `@Dao` と `@Database` の関係は？**

A: テーブルの設計図（Entity）、操作方法（DAO）、データベース全体の設定（Database）の3点セットです。Roomはこの3つから実際のデータベースコードを自動生成します。

**Q: Composeで「再コンポーズ」が多発してパフォーマンスが悪い場合は？**

A: `key` パラメータを指定するか、`remember` の使い方を見直します。でも初心者のうちは気にしなくて大丈夫。動くことの方が大事です。

---

## まとめ

- **MVVM** : UI・ロジック・データの責任を分離する設計パターン
- **Room** : SQLiteのラッパー。`@Entity`=テーブル、`@Dao`=操作、`@Database`=接続
- **Compose** : `@Composable`関数でUIを部品化。状態が変わると自動で再描画
- **Flow/StateFlow** : データが変わったら自動で通知してくれる仕組み
- **suspend** : バックグラウンドで実行される処理（UIを止めない）

完全に理解しなくていいです。「どのファイルに何が書いてあるか」が分かれば、カスタマイズできます。次の章でやってみましょう。
