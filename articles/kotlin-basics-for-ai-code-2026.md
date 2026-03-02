---
title: "AIのKotlinコードを読むための文法10選 — 非エンジニアでも理解できる"
emoji: "📖"
type: "tech"
topics: ["kotlin", "android", "jetpackcompose", "初心者"]
published: true
---

## この記事の対象

AIが生成したAndroidアプリのコードを見て「何が書いてあるかわからない」という方向け。Kotlin文法の**最低限の読み方**を10個に絞って解説します。

---

## 1. val と var

```kotlin
val name = "みょうが"  // 変更不可（定数）
var count = 0          // 変更可能（変数）
count = 1              // OK
// name = "別の名前"   // ❌ エラー
```

**AIのコードでは`val`が90%以上**。変更しないものは`val`にするのがKotlinの作法。

---

## 2. data class

```kotlin
data class Habit(
    val id: Int = 0,
    val name: String,
    val createdAt: Long
)
```

データを入れる箱。`equals()`、`toString()`、`copy()`が自動生成される。Roomの`@Entity`にも使う。

---

## 3. Null Safety（?.）

```kotlin
val name: String? = null  // ?付きはnullを許容
val length = name?.length  // nameがnullなら全体がnull
val safe = name ?: "デフォルト"  // nullなら右側の値を使う
```

Kotlinは**nullポインタ例外をコンパイル時に防ぐ**。`?`が付いた型はnullの可能性がある。

---

## 4. Lambda（ラムダ式）

```kotlin
val numbers = listOf(1, 2, 3, 4, 5)
val even = numbers.filter { it % 2 == 0 }  // [2, 4]
val doubled = numbers.map { it * 2 }        // [2, 4, 6, 8, 10]
```

`{ }` の中が関数。`it`は引数が1つの場合の省略形。Composeのコードに大量に出てくる。

---

## 5. Extension Function（拡張関数）

```kotlin
fun String.isEmail(): Boolean {
    return this.contains("@") && this.contains(".")
}

// 使い方
"test@example.com".isEmail()  // true
```

既存の型に関数を追加できる。AIがユーティリティ関数を作るときによく使う。

---

## 6. Scope Functions（let / apply / also）

```kotlin
// let: nullチェックと同時に処理
user?.let {
    println("Name: ${it.name}")
}

// apply: オブジェクトの設定をまとめて書く
val intent = Intent().apply {
    putExtra("KEY", "value")
    addFlags(Intent.FLAG_ACTIVITY_NEW_TASK)
}

// also: 副作用（ログなど）を挟む
val result = calculate().also {
    println("Result: $it")
}
```

---

## 7. when式

```kotlin
when (status) {
    "loading" -> CircularProgressIndicator()
    "error" -> Text("エラー")
    "success" -> Text("成功")
    else -> Text("不明")
}
```

Javaの`switch`の強化版。Composeの条件分岐で頻出。

---

## 8. companion object

```kotlin
class AppDatabase {
    companion object {
        private var INSTANCE: AppDatabase? = null

        fun getInstance(context: Context): AppDatabase {
            return INSTANCE ?: synchronized(this) {
                Room.databaseBuilder(context, AppDatabase::class.java, "app.db")
                    .build()
                    .also { INSTANCE = it }
            }
        }
    }
}
```

Javaの`static`に相当。シングルトン（1つだけ存在するインスタンス）を作るパターン。

---

## 9. sealed class

```kotlin
sealed class UiState {
    object Loading : UiState()
    data class Success(val data: List<Habit>) : UiState()
    data class Error(val message: String) : UiState()
}
```

「この型の子供はこれだけ」と限定する。`when`で全パターンを網羅できる。AIがUI状態管理によく使う。

---

## 10. suspend / Coroutine

```kotlin
suspend fun fetchData(): List<Habit> {
    return dao.getAllHabits()  // DBアクセス（時間がかかる処理）
}

// 呼び出し側
viewModelScope.launch {
    val data = fetchData()  // UIをブロックしない
}
```

`suspend`は「この関数は時間がかかるかもしれない」というマーク。`launch`ブロック内でのみ呼べる。

---

## まとめ

| # | 文法 | 一言解説 |
|---|------|---------|
| 1 | val/var | 定数/変数 |
| 2 | data class | データの箱 |
| 3 | ?. | null安全アクセス |
| 4 | { } | ラムダ（無名関数） |
| 5 | fun Type.name() | 拡張関数 |
| 6 | let/apply/also | スコープ関数 |
| 7 | when | 条件分岐 |
| 8 | companion object | static相当 |
| 9 | sealed class | 限定された型 |
| 10 | suspend | 非同期処理マーク |

この10個を知っていれば、AIが生成するKotlinコードの**80%は読めます**。

---

8種類のAndroidアプリテンプレート（全Kotlin + Jetpack Compose）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [AIでAndroidアプリを47秒で作った話](https://zenn.dev/myougatheaxo/articles/ai-android-app-47seconds)
- [ViewModel + StateFlow入門](https://zenn.dev/myougatheaxo/articles/android-viewmodel-stateflow-2026)
- [Kotlin Coroutine入門](https://zenn.dev/myougatheaxo/articles/kotlin-coroutine-android-basics)
- [DI（依存性注入）の選び方](https://zenn.dev/myougatheaxo/articles/android-di-manual-vs-hilt-2026)
