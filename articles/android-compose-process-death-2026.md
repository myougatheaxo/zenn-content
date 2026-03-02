---
title: "プロセス死対策完全ガイド — SavedState/Navigation/テスト手法"
emoji: "💀"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "lifecycle"]
published: true
---

## この記事で学べること

**プロセス死対策**（SavedStateHandle、Navigation状態復元、rememberSaveable、テスト方法、よくあるバグ）を解説します。

---

## プロセス死とは

```
アプリがバックグラウンド
     ↓
システムがメモリ回収のためプロセスを終了
     ↓
ユーザーがアプリに戻る
     ↓
Activity再生成（onCreate再呼び出し）
     ↓
remember → 消失 ❌
rememberSaveable → 復元 ✅
SavedStateHandle → 復元 ✅
```

---

## ViewModel + SavedStateHandle

```kotlin
@HiltViewModel
class FormViewModel @Inject constructor(
    private val savedStateHandle: SavedStateHandle
) : ViewModel() {
    // プロセス死後も自動復元
    var name by savedStateHandle.saveable { mutableStateOf("") }
        private set

    var email by savedStateHandle.saveable { mutableStateOf("") }
        private set

    val selectedTab = savedStateHandle.getStateFlow("tab", 0)

    fun updateName(value: String) { name = value; savedStateHandle["name"] = value }
    fun updateEmail(value: String) { email = value; savedStateHandle["email"] = value }
    fun selectTab(index: Int) { savedStateHandle["tab"] = index }
}
```

---

## Navigation状態復元

```kotlin
// Navigation は自動的に BackStack を SavedState に保存
// ✅ NavHost は自動でプロセス死後に画面復元

// ただし、引数は Parcelable/Serializable/プリミティブのみ
@Serializable
data class DetailRoute(
    val itemId: String,  // ✅ String → 自動保存
    val title: String    // ✅ String → 自動保存
)

// ❌ 非シリアライズ可能な引数はViewModelで管理
// val complexData: ComplexObject → SavedStateHandleで管理
```

---

## よくあるバグパターン

```kotlin
// ❌ remember: プロセス死で消失
var selectedId by remember { mutableStateOf<String?>(null) }

// ✅ rememberSaveable: プロセス死でも復元
var selectedId by rememberSaveable { mutableStateOf<String?>(null) }

// ❌ ViewModelのプロパティ: プロセス死で消失
class MyViewModel : ViewModel() {
    var count = 0  // プロセス死で0に戻る
}

// ✅ SavedStateHandle: プロセス死でも復元
class MyViewModel(savedStateHandle: SavedStateHandle) : ViewModel() {
    val count = savedStateHandle.getStateFlow("count", 0)
}
```

---

## テスト方法

```bash
# ADB でプロセス死をシミュレート
# 1. アプリを起動して操作
# 2. ホームボタンでバックグラウンドに
# 3. 以下のコマンドでプロセス強制終了
adb shell am kill com.example.app
# 4. タスク一覧からアプリに戻る
# 5. 状態が正しく復元されるか確認
```

```kotlin
// Developer Options で "Don't keep activities" を有効化
// → Activity が毎回破棄・再生成される（プロセス死の近似テスト）
```

---

## チェックリスト

```kotlin
// プロセス死対策チェックリスト:
// ✅ ユーザー入力値 → rememberSaveable or SavedStateHandle
// ✅ 選択状態 → rememberSaveable or SavedStateHandle
// ✅ スクロール位置 → LazyListState（自動保存）
// ✅ Navigation引数 → @Serializable data class
// ✅ 一時的な表示状態 → rememberSaveable
// ❌ キャッシュデータ → APIから再取得（SavedStateに入れない）
// ❌ 大きなオブジェクト → SavedStateは500KB制限
```

---

## まとめ

| 手法 | スコープ | サイズ制限 |
|------|---------|-----------|
| `remember` | Recompositionのみ | なし |
| `rememberSaveable` | Config変更+プロセス死 | Bundle制限 |
| `SavedStateHandle` | Config変更+プロセス死 | Bundle制限 |
| Room/DataStore | 永続 | なし |

- プロセス死はいつ起きてもおかしくない
- `rememberSaveable`でUI状態を確実に保存
- `SavedStateHandle`でViewModel状態を保存
- `adb shell am kill`でテスト必須

---

8種類のAndroidアプリテンプレート（プロセス死対策済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [状態保存/復元](https://zenn.dev/myougatheaxo/articles/android-compose-save-restore-state-2026)
- [Lifecycle](https://zenn.dev/myougatheaxo/articles/android-compose-lifecycle-aware-2026)
- [ViewModel](https://zenn.dev/myougatheaxo/articles/android-compose-viewmodel-hilt-2026)
