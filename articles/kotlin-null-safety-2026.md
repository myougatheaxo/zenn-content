---
title: "Kotlin Null安全完全ガイド — ?. ?: !! let の使い分け"
emoji: "🛡️"
type: "tech"
topics: ["android", "kotlin", "jetpackcompose", "初心者"]
published: true
---

## この記事で学べること

Kotlinの**Null安全**の仕組みと、`?.` `?:` `!!` `let` の正しい使い分けを解説します。

---

## Nullable型

```kotlin
// Non-null（nullを許容しない）
var name: String = "太郎"
// name = null  // コンパイルエラー

// Nullable（nullを許容する）
var nickname: String? = "たろちゃん"
nickname = null  // OK
```

---

## セーフコール (?.)

```kotlin
val length: Int? = nickname?.length
// nicknameがnullなら length もnull

// チェーンも可能
val upper: String? = user?.address?.city?.uppercase()
```

---

## エルビス演算子 (?:)

```kotlin
// nullの場合にデフォルト値
val displayName = nickname ?: "名無し"

// 早期リターン
fun processUser(user: User?) {
    val name = user?.name ?: return
    val email = user.email ?: throw IllegalArgumentException("Email required")
    // nameとemailは non-null
}
```

---

## 安全キャスト (as?)

```kotlin
val number: Int? = value as? Int
// キャストに失敗してもnull（ClassCastExceptionが起きない）
```

---

## let（nullチェック + 処理）

```kotlin
user?.let { nonNullUser ->
    updateUI(nonNullUser.name)
    saveToDatabase(nonNullUser)
}

// 変換にも
val formattedPhone: String = phone?.let { formatPhone(it) } ?: "未設定"
```

---

## !! （非推奨：強制アンラップ）

```kotlin
// NullPointerExceptionのリスクあり
val name: String = user!!.name  // userがnullならクラッシュ

// ❌ 避けるべき
fun bad(list: List<String>?) {
    list!!.forEach { println(it) }
}

// ✅ 代わりに
fun good(list: List<String>?) {
    list?.forEach { println(it) }
}
```

**`!!`はほぼ使う場面がない。**`?.`や`?:`で代替できる。

---

## Composeでのnull安全

```kotlin
@Composable
fun UserProfile(user: User?) {
    // ✅ nullチェック
    user?.let { u ->
        Column {
            Text(u.name)
            Text(u.email)
        }
    } ?: run {
        Text("ユーザーが見つかりません")
    }
}

// ViewModelの状態
@Composable
fun DetailScreen(viewModel: DetailViewModel) {
    val item by viewModel.item.collectAsStateWithLifecycle()

    item?.let { data ->
        ItemDetail(data)
    } ?: Box(Modifier.fillMaxSize(), contentAlignment = Alignment.Center) {
        CircularProgressIndicator()
    }
}
```

---

## コレクションのnull安全

```kotlin
// nullableリストの安全な操作
val items: List<String>? = getItems()
val count = items?.size ?: 0
val first = items?.firstOrNull() ?: "なし"

// リスト内のnullableアイテム
val names: List<String?> = listOf("太郎", null, "花子")
val nonNullNames: List<String> = names.filterNotNull()
```

---

## 使い分けまとめ

| 演算子 | 用途 | 例 |
|--------|------|-----|
| `?.` | nullなら処理をスキップ | `user?.name` |
| `?:` | nullならデフォルト値 | `name ?: "不明"` |
| `?.let` | nullでないとき処理実行 | `user?.let { save(it) }` |
| `as?` | 安全なキャスト | `obj as? String` |
| `!!` | 強制アンラップ（非推奨） | `user!!.name` |

---

## まとめ

- Kotlinはコンパイル時にnull安全を保証
- `?.`（セーフコール）と`?:`（エルビス）が基本
- `?.let`でnullチェック+処理
- `!!`は原則使わない
- `filterNotNull()`でリスト内のnullを除去
- Composeでは`?.let {} ?: run {}`パターン

---

8種類のAndroidアプリテンプレート（Kotlinベストプラクティス適用済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Kotlinスコープ関数ガイド](https://zenn.dev/myougatheaxo/articles/kotlin-scope-functions-2026)
- [Kotlin data class完全ガイド](https://zenn.dev/myougatheaxo/articles/kotlin-data-class-2026)
- [エラーハンドリングパターン](https://zenn.dev/myougatheaxo/articles/android-error-handling-patterns-2026)
