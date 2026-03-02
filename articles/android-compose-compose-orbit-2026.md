---
title: "Compose Orbit MVI完全ガイド — ContainerHost/Intent/SideEffect/テスト"
emoji: "🪐"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "mvi"]
published: true
---

## この記事で学べること

**Compose Orbit MVI**（ContainerHost、Intent、SideEffect、状態管理、テスト）を解説します。

---

## セットアップ

```groovy
dependencies {
    implementation("org.orbit-mvi:orbit-viewmodel:8.0.0")
    implementation("org.orbit-mvi:orbit-compose:8.0.0")
    testImplementation("org.orbit-mvi:orbit-test:8.0.0")
}
```

---

## ViewModel実装

```kotlin
data class CounterState(val count: Int = 0)

sealed class CounterSideEffect {
    data class ShowToast(val message: String) : CounterSideEffect()
}

class CounterViewModel : ViewModel(), ContainerHost<CounterState, CounterSideEffect> {
    override val container = container<CounterState, CounterSideEffect>(CounterState())

    fun increment() = intent {
        reduce { state.copy(count = state.count + 1) }
    }

    fun decrement() = intent {
        if (state.count > 0) {
            reduce { state.copy(count = state.count - 1) }
        } else {
            postSideEffect(CounterSideEffect.ShowToast("0以下にはできません"))
        }
    }

    fun reset() = intent {
        reduce { state.copy(count = 0) }
        postSideEffect(CounterSideEffect.ShowToast("リセットしました"))
    }
}
```

---

## Compose連携

```kotlin
@Composable
fun CounterScreen(viewModel: CounterViewModel = viewModel()) {
    val state by viewModel.collectAsState()
    val context = LocalContext.current

    viewModel.collectSideEffect { sideEffect ->
        when (sideEffect) {
            is CounterSideEffect.ShowToast ->
                Toast.makeText(context, sideEffect.message, Toast.LENGTH_SHORT).show()
        }
    }

    Column(
        Modifier.fillMaxSize().padding(16.dp),
        horizontalAlignment = Alignment.CenterHorizontally,
        verticalArrangement = Arrangement.Center
    ) {
        Text("${state.count}", style = MaterialTheme.typography.displayLarge)
        Spacer(Modifier.height(24.dp))
        Row(horizontalArrangement = Arrangement.spacedBy(16.dp)) {
            Button(onClick = { viewModel.decrement() }) { Text("-") }
            OutlinedButton(onClick = { viewModel.reset() }) { Text("リセット") }
            Button(onClick = { viewModel.increment() }) { Text("+") }
        }
    }
}
```

---

## テスト

```kotlin
class CounterViewModelTest {
    @Test
    fun `increment increases count`() = runTest {
        val viewModel = CounterViewModel()
        viewModel.testIntent { increment() }
        viewModel.assert(CounterState()) {
            states({ copy(count = 1) })
        }
    }

    @Test
    fun `decrement at zero shows toast`() = runTest {
        val viewModel = CounterViewModel()
        viewModel.testIntent { decrement() }
        viewModel.assert(CounterState()) {
            postedSideEffects(CounterSideEffect.ShowToast("0以下にはできません"))
        }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `ContainerHost` | ViewModel実装 |
| `intent` | アクション定義 |
| `reduce` | 状態更新 |
| `postSideEffect` | 一回限りイベント |

- `ContainerHost`でMVIパターンを簡潔に実装
- `intent`ブロック内で`reduce`と`postSideEffect`
- `collectAsState()`でCompose状態として取得
- `orbit-test`で状態遷移とSideEffectをテスト

---

8種類のAndroidアプリテンプレート（Material3対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose MVI](https://zenn.dev/myougatheaxo/articles/android-compose-compose-mvi-2026)
- [Compose UnidirectionalFlow](https://zenn.dev/myougatheaxo/articles/android-compose-compose-unidirectional-2026)
- [Compose CleanArchitecture](https://zenn.dev/myougatheaxo/articles/android-compose-compose-clean-architecture-2026)
