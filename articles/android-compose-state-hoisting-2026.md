---
title: "状態ホイスティングガイド — Composeの状態管理ベストプラクティス"
emoji: "⬆️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "state"]
published: true
---

## この記事で学べること

Composeの**状態ホイスティング**（State Hoisting）パターンとベストプラクティスを解説します。

---

## 状態ホイスティングとは

```kotlin
// ❌ 状態がコンポーネント内部に閉じている
@Composable
fun StatefulCounter() {
    var count by remember { mutableIntStateOf(0) }
    Button(onClick = { count++ }) {
        Text("Count: $count")
    }
}

// ✅ 状態をホイスティング（呼び出し側に持ち上げる）
@Composable
fun StatelessCounter(count: Int, onIncrement: () -> Unit) {
    Button(onClick = onIncrement) {
        Text("Count: $count")
    }
}

// 呼び出し側
@Composable
fun CounterScreen() {
    var count by remember { mutableIntStateOf(0) }
    StatelessCounter(count = count, onIncrement = { count++ })
}
```

---

## 実践: 検索フィールド

```kotlin
// ✅ Stateless: テスト可能、再利用可能
@Composable
fun SearchField(
    query: String,
    onQueryChange: (String) -> Unit,
    onSearch: () -> Unit,
    modifier: Modifier = Modifier
) {
    OutlinedTextField(
        value = query,
        onValueChange = onQueryChange,
        placeholder = { Text("検索...") },
        trailingIcon = {
            IconButton(onClick = onSearch) {
                Icon(Icons.Default.Search, "検索")
            }
        },
        keyboardActions = KeyboardActions(onSearch = { onSearch() }),
        singleLine = true,
        modifier = modifier.fillMaxWidth()
    )
}

// Stateful wrapper（必要な場合のみ）
@Composable
fun SearchFieldStateful(onSearch: (String) -> Unit) {
    var query by remember { mutableStateOf("") }
    SearchField(
        query = query,
        onQueryChange = { query = it },
        onSearch = { onSearch(query) }
    )
}
```

---

## 状態のレベル

```kotlin
// Level 1: Composable内 (UI状態)
@Composable
fun ExpandableCard() {
    var expanded by remember { mutableStateOf(false) } // ローカルUI状態
    Card(Modifier.clickable { expanded = !expanded }) {
        Text("Title")
        AnimatedVisibility(expanded) { Text("Details...") }
    }
}

// Level 2: Screen Composable (画面状態)
@Composable
fun ProfileScreen(viewModel: ProfileViewModel = viewModel()) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()
    // ViewModelが管理する状態
    ProfileContent(uiState, onEdit = { viewModel.edit() })
}

// Level 3: ViewModel (ビジネスロジック)
class ProfileViewModel : ViewModel() {
    private val _uiState = MutableStateFlow(ProfileUiState())
    val uiState: StateFlow<ProfileUiState> = _uiState.asStateFlow()
}
```

---

## いつホイスティングするか

```
ホイスティングする:
  ✅ 複数のコンポーネントで共有する状態
  ✅ テストしたいビジネスロジック
  ✅ 親が制御する必要がある状態

ホイスティングしない:
  ❌ 純粋にUI内部の状態（展開/折りたたみ等）
  ❌ 他のコンポーネントが参照しない状態
```

---

## まとめ

- 状態ホイスティング: 状態を上位に持ち上げるパターン
- Statelessコンポーネント: `value` + `onValueChange`パラメータ
- テスト可能性と再利用性が向上
- UI状態はローカル`remember`、ビジネスロジックはViewModel
- 単一の真実の情報源（Single Source of Truth）
- 状態を共有する最も近い共通の祖先にホイスティング

---

8種類のAndroidアプリテンプレート（状態管理設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [状態管理ガイド](https://zenn.dev/myougatheaxo/articles/compose-state-management-2026)
- [rememberパターンガイド](https://zenn.dev/myougatheaxo/articles/android-compose-remember-patterns-2026)
- [ViewModel/StateFlowガイド](https://zenn.dev/myougatheaxo/articles/android-viewmodel-stateflow-2026)
