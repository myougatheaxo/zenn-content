---
title: "Kotlinスコープ関数完全ガイド — let/run/with/apply/also使い分け"
emoji: "🔧"
type: "tech"
topics: ["android", "kotlin", "jetpackcompose", "初心者"]
published: true
---

## この記事で学べること

Kotlinの5つのスコープ関数 **let, run, with, apply, also** の違いと使い分けを解説します。

---

## 一覧表

| 関数 | レシーバ | 戻り値 | 主な用途 |
|------|---------|--------|---------|
| `let` | `it` | ラムダの結果 | null安全呼び出し |
| `run` | `this` | ラムダの結果 | 初期化+計算 |
| `with` | `this` | ラムダの結果 | オブジェクト操作 |
| `apply` | `this` | オブジェクト自身 | 初期化・設定 |
| `also` | `it` | オブジェクト自身 | 副作用（ログ等） |

---

## let — null安全チェック

```kotlin
// nullableの安全な処理
val name: String? = getUserName()

name?.let { nonNullName ->
    println("名前: $nonNullName")
    updateUI(nonNullName)
}

// 変換にも使える
val length = name?.let { it.length } ?: 0
```

**使い場面**: `?.let` でnullチェック、変数のスコープ限定

---

## run — 初期化 + 結果を返す

```kotlin
val result = service.run {
    port = 8080
    host = "localhost"
    connect() // 戻り値
}

// 拡張関数なしでも使える
val hexColor = run {
    val red = 255
    val green = 128
    val blue = 0
    "#%02x%02x%02x".format(red, green, blue)
}
```

**使い場面**: オブジェクトの設定後に結果を計算

---

## with — 既存オブジェクトの操作

```kotlin
val person = Person("太郎", 25)

val description = with(person) {
    "名前: $name, 年齢: $age"
}

// Canvas描画でよく使う
with(canvas) {
    drawColor(Color.WHITE)
    drawCircle(100f, 100f, 50f, paint)
    drawText("Hello", 50f, 200f, paint)
}
```

**使い場面**: 既存オブジェクトの複数プロパティにアクセス

---

## apply — オブジェクトの初期化

```kotlin
val textView = TextView(context).apply {
    text = "Hello"
    textSize = 16f
    setTextColor(Color.BLACK)
    setPadding(16, 8, 16, 8)
}

// Intent作成
val intent = Intent(context, DetailActivity::class.java).apply {
    putExtra("id", itemId)
    putExtra("title", itemTitle)
    addFlags(Intent.FLAG_ACTIVITY_NEW_TASK)
}
```

**使い場面**: ビルダーパターンの代替、オブジェクト設定

---

## also — 副作用（ログ、デバッグ）

```kotlin
val numbers = mutableListOf(1, 2, 3)
    .also { println("初期リスト: $it") }
    .also { it.add(4) }
    .also { println("追加後: $it") }

// チェーンの途中でログ
fun getUser(id: Int): User =
    repository.findById(id)
        .also { Log.d("TAG", "取得したユーザー: $it") }
```

**使い場面**: デバッグログ、バリデーション、チェーン中の副作用

---

## 実践的な組み合わせ

```kotlin
// ネットワークレスポンスの処理
suspend fun fetchUser(id: Int): Result<User> = runCatching {
    api.getUser(id)
}.also { result ->
    Log.d("API", "fetchUser($id): ${result.isSuccess}")
}.map { response ->
    response.body()?.let { dto ->
        dto.toUser()
    } ?: throw IllegalStateException("Empty body")
}
```

```kotlin
// SharedPreferences
context.getSharedPreferences("app", MODE_PRIVATE)
    .edit()
    .apply {
        putString("token", token)
        putBoolean("loggedIn", true)
    }
    .apply() // SharedPreferences.Editor.apply()
```

---

## 判断フローチャート

1. **nullチェック？** → `let`
2. **オブジェクト初期化？** → `apply`
3. **ログ・デバッグ？** → `also`
4. **設定+結果が必要？** → `run`
5. **既存オブジェクトの複数操作？** → `with`

---

## まとめ

- **`let`**: `?.let` でnull安全処理
- **`apply`**: オブジェクト初期化（Intent, View, Builder）
- **`also`**: 副作用（ログ、デバッグ）
- **`run`**: 初期化+計算結果
- **`with`**: 既存オブジェクトへの複数アクセス
- 迷ったら `let`（null処理）か `apply`（初期化）

---

8種類のAndroidアプリテンプレート（Kotlinベストプラクティス適用済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Kotlin Extension Functions活用術](https://zenn.dev/myougatheaxo/articles/kotlin-extension-functions-2026)
- [Kotlin Sealed Class完全ガイド](https://zenn.dev/myougatheaxo/articles/kotlin-sealed-class-2026)
- [Kotlin Coroutines & Flow入門](https://zenn.dev/myougatheaxo/articles/kotlin-coroutine-flow-2026)
