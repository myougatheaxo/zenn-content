---
title: "Compose rememberSaveable完全ガイド — 状態保存/Saver/listSaver/mapSaver"
emoji: "💾"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "state"]
published: true
---

## この記事で学べること

**Compose rememberSaveable**（状態保存、カスタムSaver、listSaver、mapSaver、Bundle互換型）を解説します。

---

## remember vs rememberSaveable

```kotlin
@Composable
fun ComparisonDemo() {
    // remember: 設定変更（画面回転）で消える
    var rememberCount by remember { mutableIntStateOf(0) }

    // rememberSaveable: 設定変更+プロセス終了でも保持
    var saveableCount by rememberSaveable { mutableIntStateOf(0) }

    Column(Modifier.padding(16.dp), verticalArrangement = Arrangement.spacedBy(8.dp)) {
        Text("remember: $rememberCount")
        Button(onClick = { rememberCount++ }) { Text("remember++") }

        Text("rememberSaveable: $saveableCount")
        Button(onClick = { saveableCount++ }) { Text("saveable++") }
    }
}
```

---

## listSaver

```kotlin
@Parcelize
data class UserProfile(val name: String, val age: Int, val email: String) : Parcelable

// listSaverで保存
val UserProfileSaver = listSaver<UserProfile, Any>(
    save = { listOf(it.name, it.age, it.email) },
    restore = { UserProfile(it[0] as String, it[1] as Int, it[2] as String) }
)

@Composable
fun ProfileEditor() {
    var profile by rememberSaveable(stateSaver = UserProfileSaver) {
        mutableStateOf(UserProfile("", 0, ""))
    }

    Column(Modifier.padding(16.dp), verticalArrangement = Arrangement.spacedBy(8.dp)) {
        OutlinedTextField(
            value = profile.name,
            onValueChange = { profile = profile.copy(name = it) },
            label = { Text("名前") }
        )
        OutlinedTextField(
            value = profile.age.toString(),
            onValueChange = { profile = profile.copy(age = it.toIntOrNull() ?: 0) },
            label = { Text("年齢") }
        )
        OutlinedTextField(
            value = profile.email,
            onValueChange = { profile = profile.copy(email = it) },
            label = { Text("メール") }
        )
    }
}
```

---

## Parcelable（最も簡単）

```kotlin
@Parcelize
data class FormState(
    val title: String = "",
    val description: String = "",
    val priority: Int = 0
) : Parcelable

@Composable
fun FormWithParcelable() {
    // Parcelableなら自動保存
    var state by rememberSaveable { mutableStateOf(FormState()) }

    Column(Modifier.padding(16.dp), verticalArrangement = Arrangement.spacedBy(8.dp)) {
        OutlinedTextField(
            value = state.title,
            onValueChange = { state = state.copy(title = it) },
            label = { Text("タイトル") }
        )
        OutlinedTextField(
            value = state.description,
            onValueChange = { state = state.copy(description = it) },
            label = { Text("説明") }
        )
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `rememberSaveable` | 設定変更/プロセス終了対応 |
| `Parcelable` | 最も簡単な保存方法 |
| `listSaver` | リスト形式で保存 |
| `mapSaver` | マップ形式で保存 |

- `rememberSaveable`は`remember`の永続版
- `@Parcelize`を使えば自動でBundle保存
- 非Parcelableは`listSaver`/`mapSaver`でカスタム保存
- Bundle制限: 大きなデータは保存しない（512KB上限）

---

8種類のAndroidアプリテンプレート（状態管理対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose ProcessDeath](https://zenn.dev/myougatheaxo/articles/android-compose-compose-process-death-2026)
- [Compose DerivedState](https://zenn.dev/myougatheaxo/articles/android-compose-compose-derived-state-2026)
- [Compose SideEffect](https://zenn.dev/myougatheaxo/articles/android-compose-compose-side-effect-2026)
