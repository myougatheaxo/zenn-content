---
title: "状態マシン設計パターン完全ガイド — sealed class/UiState/イベント駆動"
emoji: "🔄"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "architecture"]
published: true
---

## この記事で学べること

**状態マシン設計**（sealed classベースの状態遷移、イベント駆動、副作用管理、テスタブルな状態管理）を解説します。

---

## UiState設計

```kotlin
sealed interface OrderUiState {
    data object Idle : OrderUiState
    data object Loading : OrderUiState
    data class Cart(val items: List<CartItem>, val total: Int) : OrderUiState
    data class Confirming(val order: Order) : OrderUiState
    data class Processing(val orderId: String) : OrderUiState
    data class Completed(val orderId: String, val estimatedDelivery: String) : OrderUiState
    data class Error(val message: String, val retryAction: OrderEvent?) : OrderUiState
}
```

---

## イベント定義

```kotlin
sealed interface OrderEvent {
    data class AddItem(val item: Product, val quantity: Int) : OrderEvent
    data class RemoveItem(val itemId: String) : OrderEvent
    data object Checkout : OrderEvent
    data object ConfirmOrder : OrderEvent
    data object CancelOrder : OrderEvent
    data object Retry : OrderEvent
}
```

---

## ViewModel（状態マシン）

```kotlin
@HiltViewModel
class OrderViewModel @Inject constructor(
    private val orderRepository: OrderRepository,
    private val cartRepository: CartRepository
) : ViewModel() {

    private val _uiState = MutableStateFlow<OrderUiState>(OrderUiState.Idle)
    val uiState: StateFlow<OrderUiState> = _uiState

    private val _sideEffects = Channel<OrderSideEffect>(Channel.BUFFERED)
    val sideEffects = _sideEffects.receiveAsFlow()

    fun onEvent(event: OrderEvent) {
        val currentState = _uiState.value
        when (event) {
            is OrderEvent.AddItem -> handleAddItem(currentState, event)
            is OrderEvent.RemoveItem -> handleRemoveItem(currentState, event)
            OrderEvent.Checkout -> handleCheckout(currentState)
            OrderEvent.ConfirmOrder -> handleConfirm(currentState)
            OrderEvent.CancelOrder -> _uiState.value = OrderUiState.Idle
            OrderEvent.Retry -> handleRetry(currentState)
        }
    }

    private fun handleCheckout(state: OrderUiState) {
        if (state !is OrderUiState.Cart) return
        if (state.items.isEmpty()) {
            _sideEffects.trySend(OrderSideEffect.ShowMessage("カートが空です"))
            return
        }
        _uiState.value = OrderUiState.Confirming(Order(items = state.items, total = state.total))
    }

    private fun handleConfirm(state: OrderUiState) {
        if (state !is OrderUiState.Confirming) return
        viewModelScope.launch {
            _uiState.value = OrderUiState.Processing(state.order.id)
            when (val result = orderRepository.placeOrder(state.order)) {
                is AppResult.Success -> {
                    _uiState.value = OrderUiState.Completed(result.data.id, result.data.estimatedDelivery)
                    _sideEffects.send(OrderSideEffect.ShowMessage("注文が完了しました"))
                }
                is AppResult.Error -> {
                    _uiState.value = OrderUiState.Error("注文に失敗しました", OrderEvent.ConfirmOrder)
                }
            }
        }
    }
}

sealed interface OrderSideEffect {
    data class ShowMessage(val message: String) : OrderSideEffect
    data class Navigate(val route: String) : OrderSideEffect
}
```

---

## Compose画面

```kotlin
@Composable
fun OrderScreen(viewModel: OrderViewModel = hiltViewModel()) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()
    val snackbarHostState = remember { SnackbarHostState() }

    LaunchedEffect(Unit) {
        viewModel.sideEffects.collect { effect ->
            when (effect) {
                is OrderSideEffect.ShowMessage -> snackbarHostState.showSnackbar(effect.message)
                is OrderSideEffect.Navigate -> { /* navigate */ }
            }
        }
    }

    Scaffold(snackbarHost = { SnackbarHost(snackbarHostState) }) { padding ->
        when (val state = uiState) {
            OrderUiState.Idle -> EmptyCartContent(Modifier.padding(padding))
            OrderUiState.Loading -> LoadingContent()
            is OrderUiState.Cart -> CartContent(state, viewModel::onEvent)
            is OrderUiState.Confirming -> ConfirmContent(state, viewModel::onEvent)
            is OrderUiState.Processing -> ProcessingContent(state)
            is OrderUiState.Completed -> CompletedContent(state)
            is OrderUiState.Error -> ErrorContent(state, viewModel::onEvent)
        }
    }
}
```

---

## まとめ

| 要素 | 役割 |
|------|------|
| UiState | 画面の状態（sealed interface） |
| Event | ユーザーアクション |
| SideEffect | 一回限りのイベント（Snackbar等） |
| ViewModel | 状態遷移ロジック |

- sealed interfaceで全状態を型安全に定義
- `when`式でコンパイラが網羅性チェック
- SideEffectでSnackbar/Navigationを一回だけ実行
- 不正な状態遷移を型レベルで防止

---

8種類のAndroidアプリテンプレート（状態マシン設計）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [State管理](https://zenn.dev/myougatheaxo/articles/android-compose-state-management-2026)
- [Result/Eitherパターン](https://zenn.dev/myougatheaxo/articles/android-compose-result-api-pattern-2026)
- [Circuit/Moleculeパターン](https://zenn.dev/myougatheaxo/articles/android-compose-circuit-pattern-2026)
