---
title: "State Hoisting完全ガイド — 状態の巻き上げ/Stateless設計/UDF"
emoji: "⬆️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "architecture"]
published: true
---

## この記事で学べること

**State Hoisting**（状態の巻き上げ、Statelessコンポーザブル設計、単方向データフロー）を解説します。

---

## State Hoistingの基本

```kotlin
// ❌ Stateful（テスト困難、再利用不可）
@Composable
fun StatefulCounter() {
    var count by remember { mutableIntStateOf(0) }
    Button(onClick = { count++ }) {
        Text("Count: $count")
    }
}

// ✅ Stateless（状態を呼び出し元に巻き上げ）
@Composable
fun StatelessCounter(
    count: Int,
    onIncrement: () -> Unit,
    modifier: Modifier = Modifier
) {
    Button(onClick = onIncrement, modifier = modifier) {
        Text("Count: $count")
    }
}

// 呼び出し元で状態管理
@Composable
fun CounterScreen() {
    var count by remember { mutableIntStateOf(0) }
    StatelessCounter(
        count = count,
        onIncrement = { count++ }
    )
}
```

---

## フォームでのState Hoisting

```kotlin
@Composable
fun LoginForm(
    email: String,
    password: String,
    onEmailChange: (String) -> Unit,
    onPasswordChange: (String) -> Unit,
    onLogin: () -> Unit,
    isLoading: Boolean,
    modifier: Modifier = Modifier
) {
    Column(modifier.padding(16.dp)) {
        OutlinedTextField(
            value = email,
            onValueChange = onEmailChange,
            label = { Text("メール") },
            modifier = Modifier.fillMaxWidth()
        )
        Spacer(Modifier.height(8.dp))
        OutlinedTextField(
            value = password,
            onValueChange = onPasswordChange,
            label = { Text("パスワード") },
            visualTransformation = PasswordVisualTransformation(),
            modifier = Modifier.fillMaxWidth()
        )
        Spacer(Modifier.height(16.dp))
        Button(
            onClick = onLogin,
            enabled = !isLoading,
            modifier = Modifier.fillMaxWidth()
        ) {
            if (isLoading) CircularProgressIndicator(Modifier.size(20.dp))
            else Text("ログイン")
        }
    }
}

@Composable
fun LoginScreen(viewModel: LoginViewModel = hiltViewModel()) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()

    LoginForm(
        email = uiState.email,
        password = uiState.password,
        onEmailChange = viewModel::updateEmail,
        onPasswordChange = viewModel::updatePassword,
        onLogin = viewModel::login,
        isLoading = uiState.isLoading
    )
}
```

---

## State Holderパターン

```kotlin
class SearchBarState(
    initialQuery: String = "",
    private val onSearch: (String) -> Unit
) {
    var query by mutableStateOf(initialQuery)
        private set

    var isActive by mutableStateOf(false)
        private set

    fun updateQuery(newQuery: String) {
        query = newQuery
    }

    fun activate() { isActive = true }
    fun deactivate() { isActive = false; query = "" }
    fun submit() { onSearch(query) }
}

@Composable
fun rememberSearchBarState(
    initialQuery: String = "",
    onSearch: (String) -> Unit
): SearchBarState = remember {
    SearchBarState(initialQuery, onSearch)
}

@Composable
fun SearchBar(state: SearchBarState, modifier: Modifier = Modifier) {
    OutlinedTextField(
        value = state.query,
        onValueChange = state::updateQuery,
        placeholder = { Text("検索...") },
        trailingIcon = {
            IconButton(onClick = state::submit) {
                Icon(Icons.Default.Search, "検索")
            }
        },
        modifier = modifier.fillMaxWidth()
    )
}
```

---

## まとめ

| パターン | 用途 |
|---------|------|
| State Hoisting | 状態を呼び出し元に巻き上げ |
| Stateless Composable | テスト・再利用しやすいUI |
| State Holder | 複数状態をまとめて管理 |
| UDF | ViewModel→UI→Event→ViewModel |

- Statelessコンポーザブルはテスト・プレビューが容易
- 状態は「最も低い共通の親」で保持
- State Holderで関連状態をカプセル化
- ViewModelはScreen単位の状態管理に使用

---

8種類のAndroidアプリテンプレート（アーキテクチャ実装済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [State Management](https://zenn.dev/myougatheaxo/articles/android-compose-compose-state-management-2026)
- [Recomposition](https://zenn.dev/myougatheaxo/articles/android-compose-compose-recomposition-2026)
- [Side Effects](https://zenn.dev/myougatheaxo/articles/android-compose-compose-side-effects-2026)
