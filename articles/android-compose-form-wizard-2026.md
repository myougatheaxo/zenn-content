---
title: "フォームウィザード実装ガイド — ステップ制フォーム/プログレス表示"
emoji: "📝"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "forms"]
published: true
---

## この記事で学べること

Composeでの**ステップ制フォーム（ウィザード）**の実装を解説します。

---

## ステップ定義

```kotlin
enum class WizardStep(val title: String) {
    PERSONAL("個人情報"),
    ADDRESS("住所"),
    CONFIRM("確認")
}

data class WizardState(
    val currentStep: WizardStep = WizardStep.PERSONAL,
    val name: String = "",
    val email: String = "",
    val phone: String = "",
    val address: String = "",
    val city: String = "",
    val zipCode: String = ""
)
```

---

## プログレスインジケーター

```kotlin
@Composable
fun StepIndicator(currentStep: Int, totalSteps: Int) {
    Row(
        Modifier.fillMaxWidth().padding(16.dp),
        horizontalArrangement = Arrangement.SpaceBetween,
        verticalAlignment = Alignment.CenterVertically
    ) {
        for (i in 0 until totalSteps) {
            // ステップ番号
            Box(
                Modifier
                    .size(32.dp)
                    .clip(CircleShape)
                    .background(
                        if (i <= currentStep) MaterialTheme.colorScheme.primary
                        else MaterialTheme.colorScheme.surfaceVariant
                    ),
                contentAlignment = Alignment.Center
            ) {
                if (i < currentStep) {
                    Icon(Icons.Default.Check, null, tint = Color.White, modifier = Modifier.size(16.dp))
                } else {
                    Text(
                        "${i + 1}",
                        color = if (i == currentStep) Color.White else MaterialTheme.colorScheme.onSurfaceVariant,
                        style = MaterialTheme.typography.labelMedium
                    )
                }
            }
            // 接続線
            if (i < totalSteps - 1) {
                HorizontalDivider(
                    Modifier.weight(1f).padding(horizontal = 8.dp),
                    color = if (i < currentStep) MaterialTheme.colorScheme.primary
                            else MaterialTheme.colorScheme.surfaceVariant,
                    thickness = 2.dp
                )
            }
        }
    }
}
```

---

## ウィザード画面

```kotlin
@Composable
fun FormWizard(viewModel: WizardViewModel = viewModel()) {
    val state by viewModel.state.collectAsStateWithLifecycle()
    val steps = WizardStep.entries
    val currentIndex = steps.indexOf(state.currentStep)

    Scaffold(
        topBar = {
            TopAppBar(title = { Text(state.currentStep.title) })
        },
        bottomBar = {
            Row(
                Modifier.fillMaxWidth().padding(16.dp),
                horizontalArrangement = Arrangement.SpaceBetween
            ) {
                OutlinedButton(
                    onClick = { viewModel.previousStep() },
                    enabled = currentIndex > 0
                ) { Text("戻る") }

                Button(
                    onClick = {
                        if (currentIndex == steps.lastIndex) viewModel.submit()
                        else viewModel.nextStep()
                    }
                ) {
                    Text(if (currentIndex == steps.lastIndex) "送信" else "次へ")
                }
            }
        }
    ) { padding ->
        Column(Modifier.padding(padding)) {
            StepIndicator(currentIndex, steps.size)

            AnimatedContent(
                targetState = state.currentStep,
                transitionSpec = {
                    slideIntoContainer(AnimatedContentTransitionScope.SlideDirection.Left) togetherWith
                    slideOutOfContainer(AnimatedContentTransitionScope.SlideDirection.Left)
                },
                label = "wizard"
            ) { step ->
                when (step) {
                    WizardStep.PERSONAL -> PersonalStep(state, viewModel)
                    WizardStep.ADDRESS -> AddressStep(state, viewModel)
                    WizardStep.CONFIRM -> ConfirmStep(state)
                }
            }
        }
    }
}
```

---

## 各ステップの実装

```kotlin
@Composable
fun PersonalStep(state: WizardState, viewModel: WizardViewModel) {
    Column(Modifier.padding(16.dp)) {
        OutlinedTextField(
            value = state.name,
            onValueChange = { viewModel.updateName(it) },
            label = { Text("名前") },
            modifier = Modifier.fillMaxWidth()
        )
        Spacer(Modifier.height(8.dp))
        OutlinedTextField(
            value = state.email,
            onValueChange = { viewModel.updateEmail(it) },
            label = { Text("メール") },
            keyboardOptions = KeyboardOptions(keyboardType = KeyboardType.Email),
            modifier = Modifier.fillMaxWidth()
        )
        Spacer(Modifier.height(8.dp))
        OutlinedTextField(
            value = state.phone,
            onValueChange = { viewModel.updatePhone(it) },
            label = { Text("電話番号") },
            keyboardOptions = KeyboardOptions(keyboardType = KeyboardType.Phone),
            modifier = Modifier.fillMaxWidth()
        )
    }
}

@Composable
fun ConfirmStep(state: WizardState) {
    Column(Modifier.padding(16.dp)) {
        Text("入力内容の確認", style = MaterialTheme.typography.titleMedium)
        Spacer(Modifier.height(16.dp))
        ConfirmRow("名前", state.name)
        ConfirmRow("メール", state.email)
        ConfirmRow("電話", state.phone)
        ConfirmRow("住所", state.address)
        ConfirmRow("市区町村", state.city)
        ConfirmRow("郵便番号", state.zipCode)
    }
}

@Composable
fun ConfirmRow(label: String, value: String) {
    Row(Modifier.fillMaxWidth().padding(vertical = 4.dp)) {
        Text(label, Modifier.width(80.dp), color = MaterialTheme.colorScheme.outline)
        Text(value)
    }
}
```

---

## まとめ

- `enum class`でステップを定義
- `StepIndicator`でプログレス表示（番号/チェック/接続線）
- `AnimatedContent`でステップ切り替えアニメーション
- Scaffold `bottomBar`で戻る/次へボタン
- 確認ステップで入力内容を一覧表示
- ViewModelで状態管理とバリデーション

---

8種類のAndroidアプリテンプレート（フォーム設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [入力フォーム完全ガイド](https://zenn.dev/myougatheaxo/articles/compose-forms-input-2026)
- [入力バリデーションガイド](https://zenn.dev/myougatheaxo/articles/android-compose-text-field-validation-2026)
- [Stepper/Sliderガイド](https://zenn.dev/myougatheaxo/articles/android-compose-stepper-slider-2026)
