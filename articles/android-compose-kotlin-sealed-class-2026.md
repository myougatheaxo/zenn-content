---
title: "Kotlin Sealed Class完全ガイド — sealed class/sealed interface/when式/UiState"
emoji: "🔒"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "sealed"]
published: true
---

## この記事で学べること

**Kotlin Sealed Class**（sealed class、sealed interface、exhaustive when、UiState設計パターン）を解説します。

---

## sealed class基本

```kotlin
sealed class Result<out T> {
    data class Success<T>(val data: T) : Result<T>()
    data class Error(val exception: Throwable) : Result<Nothing>()
    data object Loading : Result<Nothing>()
}

// exhaustive when（全パターンを網羅）
fun <T> handleResult(result: Result<T>) {
    when (result) {
        is Result.Success -> println("成功: ${result.data}")
        is Result.Error -> println("エラー: ${result.exception.message}")
        Result.Loading -> println("読み込み中")
        // else不要（コンパイラが網羅性をチェック）
    }
}
```

---

## sealed interface

```kotlin
// sealed interfaceは複数継承可能
sealed interface UiEvent
sealed interface NavigationEvent : UiEvent

data class ShowSnackbar(val message: String) : UiEvent
data class NavigateTo(val route: String) : NavigationEvent
data object NavigateBack : NavigationEvent

// Compose UIでの使用
sealed interface ScreenState {
    data object Idle : ScreenState
    data object Loading : ScreenState
    data class Content(val items: List<Item>) : ScreenState
    data class Error(val message: String, val retry: Boolean = true) : ScreenState
}

@Composable
fun ScreenContent(state: ScreenState) {
    when (state) {
        ScreenState.Idle -> {} // 何も表示しない
        ScreenState.Loading -> CircularProgressIndicator()
        is ScreenState.Content -> {
            LazyColumn {
                items(state.items) { Text(it.name) }
            }
        }
        is ScreenState.Error -> {
            Column(horizontalAlignment = Alignment.CenterHorizontally) {
                Text(state.message, color = Color.Red)
                if (state.retry) {
                    Button(onClick = { /* retry */ }) { Text("再試行") }
                }
            }
        }
    }
}
```

---

## 実践パターン

```kotlin
// ナビゲーションルート
sealed class Screen(val route: String) {
    data object Home : Screen("home")
    data object Settings : Screen("settings")
    data class Detail(val id: String) : Screen("detail/$id")
}

// APIレスポンスのマッピング
sealed class ApiResult<out T> {
    data class Success<T>(val data: T) : ApiResult<T>()
    data class HttpError(val code: Int, val message: String) : ApiResult<Nothing>()
    data class NetworkError(val exception: IOException) : ApiResult<Nothing>()
    data object Unauthorized : ApiResult<Nothing>()
}

fun <T> ApiResult<T>.fold(
    onSuccess: (T) -> Unit,
    onError: (String) -> Unit
) = when (this) {
    is ApiResult.Success -> onSuccess(data)
    is ApiResult.HttpError -> onError("HTTP $code: $message")
    is ApiResult.NetworkError -> onError("ネットワークエラー")
    ApiResult.Unauthorized -> onError("認証エラー")
}
```

---

## まとめ

| 概念 | 用途 |
|------|------|
| `sealed class` | 制限付き継承 |
| `sealed interface` | 制限付きインターフェース |
| exhaustive when | 網羅性チェック |
| `data object` | シングルトンサブクラス |

- `sealed class`でサブクラスを同ファイル内に制限
- `when`式でコンパイル時に網羅性チェック
- UiState/Result/Eventの設計に最適
- `sealed interface`で複数インターフェースの実装が可能

---

8種類のAndroidアプリテンプレート（Kotlin対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose MVI](https://zenn.dev/myougatheaxo/articles/android-compose-compose-mvi-2026)
- [Kotlin ValueClass](https://zenn.dev/myougatheaxo/articles/android-compose-kotlin-value-class-2026)
- [Compose UnidirectionalFlow](https://zenn.dev/myougatheaxo/articles/android-compose-compose-unidirectional-2026)
