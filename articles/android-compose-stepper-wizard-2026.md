---
title: "ステッパー/ウィザード完全ガイド — 複数ステップフォーム/進捗表示/バリデーション"
emoji: "📊"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "ui"]
published: true
---

## この記事で学べること

**ステッパー/ウィザード**（複数ステップUI、進捗インジケーター、ステップバリデーション、HorizontalPager連携）を解説します。

---

## ステッパーインジケーター

```kotlin
@Composable
fun StepIndicator(
    totalSteps: Int,
    currentStep: Int,
    modifier: Modifier = Modifier
) {
    Row(
        modifier.fillMaxWidth().padding(horizontal = 16.dp),
        horizontalArrangement = Arrangement.SpaceBetween,
        verticalAlignment = Alignment.CenterVertically
    ) {
        repeat(totalSteps) { index ->
            val isCompleted = index < currentStep
            val isCurrent = index == currentStep

            // ステップ番号
            Box(
                Modifier
                    .size(32.dp)
                    .background(
                        when {
                            isCompleted -> MaterialTheme.colorScheme.primary
                            isCurrent -> MaterialTheme.colorScheme.primaryContainer
                            else -> MaterialTheme.colorScheme.surfaceVariant
                        },
                        CircleShape
                    ),
                contentAlignment = Alignment.Center
            ) {
                if (isCompleted) {
                    Icon(Icons.Default.Check, null, tint = MaterialTheme.colorScheme.onPrimary, modifier = Modifier.size(16.dp))
                } else {
                    Text("${index + 1}", color = if (isCurrent) MaterialTheme.colorScheme.onPrimaryContainer else MaterialTheme.colorScheme.onSurfaceVariant, fontSize = 14.sp)
                }
            }

            // 接続線
            if (index < totalSteps - 1) {
                Box(
                    Modifier
                        .weight(1f)
                        .height(2.dp)
                        .padding(horizontal = 4.dp)
                        .background(
                            if (isCompleted) MaterialTheme.colorScheme.primary
                            else MaterialTheme.colorScheme.surfaceVariant
                        )
                )
            }
        }
    }
}
```

---

## ウィザードフォーム

```kotlin
@Composable
fun WizardForm(viewModel: WizardViewModel = hiltViewModel()) {
    val currentStep by viewModel.currentStep.collectAsStateWithLifecycle()
    val steps = listOf("基本情報", "連絡先", "確認")

    Column(Modifier.fillMaxSize()) {
        StepIndicator(totalSteps = steps.size, currentStep = currentStep)

        Spacer(Modifier.height(16.dp))
        Text(steps[currentStep], style = MaterialTheme.typography.titleLarge, modifier = Modifier.padding(horizontal = 16.dp))

        // ステップコンテンツ
        Box(Modifier.weight(1f).padding(16.dp)) {
            when (currentStep) {
                0 -> BasicInfoStep(viewModel)
                1 -> ContactStep(viewModel)
                2 -> ConfirmStep(viewModel)
            }
        }

        // ナビゲーションボタン
        Row(Modifier.fillMaxWidth().padding(16.dp), horizontalArrangement = Arrangement.SpaceBetween) {
            if (currentStep > 0) {
                OutlinedButton(onClick = { viewModel.previousStep() }) { Text("戻る") }
            } else {
                Spacer(Modifier.width(1.dp))
            }

            if (currentStep < steps.lastIndex) {
                Button(onClick = { viewModel.nextStep() }, enabled = viewModel.isCurrentStepValid()) {
                    Text("次へ")
                }
            } else {
                Button(onClick = { viewModel.submit() }) { Text("送信") }
            }
        }
    }
}
```

---

## ViewModel

```kotlin
@HiltViewModel
class WizardViewModel @Inject constructor() : ViewModel() {
    val currentStep = MutableStateFlow(0)

    var name by mutableStateOf("")
    var email by mutableStateOf("")
    var phone by mutableStateOf("")

    fun nextStep() {
        if (isCurrentStepValid()) currentStep.update { it + 1 }
    }

    fun previousStep() { currentStep.update { maxOf(0, it - 1) } }

    fun isCurrentStepValid(): Boolean = when (currentStep.value) {
        0 -> name.isNotBlank()
        1 -> email.contains("@") && phone.isNotBlank()
        else -> true
    }

    fun submit() { /* 送信処理 */ }
}
```

---

## まとめ

| 要素 | 実装 |
|------|------|
| インジケーター | `StepIndicator` |
| コンテンツ切替 | `when(currentStep)` |
| バリデーション | ステップ毎の検証 |
| ナビゲーション | 前へ/次へボタン |

- ステッパーインジケーターで進捗を視覚化
- ステップ毎のバリデーションで入力品質を担保
- `when`式でステップコンテンツを切替
- 戻るボタンで前ステップに修正可能

---

8種類のAndroidアプリテンプレート（フォームUI実装済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [フォームバリデーション](https://zenn.dev/myougatheaxo/articles/android-compose-form-validation-2026)
- [TabRow/Pager](https://zenn.dev/myougatheaxo/articles/android-compose-tabrow-viewpager-2026)
- [状態管理](https://zenn.dev/myougatheaxo/articles/android-compose-state-management-2026)
