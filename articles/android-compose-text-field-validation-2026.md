---
title: "入力バリデーションガイド — リアルタイム検証/エラー表示/送信制御"
emoji: "✏️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "forms"]
published: true
---

## この記事で学べること

Composeでの**入力バリデーション**（リアルタイム検証、エラー表示、送信ボタン制御）を解説します。

---

## 基本のバリデーション

```kotlin
@Composable
fun ValidatedTextField() {
    var email by remember { mutableStateOf("") }
    var isError by remember { mutableStateOf(false) }
    var errorMessage by remember { mutableStateOf("") }

    OutlinedTextField(
        value = email,
        onValueChange = {
            email = it
            val result = validateEmail(it)
            isError = !result.isValid
            errorMessage = result.message
        },
        label = { Text("メールアドレス") },
        isError = isError,
        supportingText = {
            if (isError) {
                Text(errorMessage, color = MaterialTheme.colorScheme.error)
            }
        },
        trailingIcon = {
            if (isError) {
                Icon(Icons.Default.Error, "エラー", tint = MaterialTheme.colorScheme.error)
            }
        },
        keyboardOptions = KeyboardOptions(keyboardType = KeyboardType.Email),
        modifier = Modifier.fillMaxWidth()
    )
}

data class ValidationResult(val isValid: Boolean, val message: String = "")

fun validateEmail(email: String): ValidationResult {
    return when {
        email.isBlank() -> ValidationResult(false, "メールアドレスを入力してください")
        !email.contains("@") -> ValidationResult(false, "正しいメールアドレスを入力してください")
        !email.matches(Regex(".+@.+\\..+")) -> ValidationResult(false, "メールアドレスの形式が正しくありません")
        else -> ValidationResult(true)
    }
}
```

---

## フォーム全体のバリデーション

```kotlin
data class FormState(
    val name: String = "",
    val email: String = "",
    val password: String = "",
    val nameError: String? = null,
    val emailError: String? = null,
    val passwordError: String? = null
) {
    val isValid: Boolean get() = nameError == null && emailError == null &&
        passwordError == null && name.isNotBlank() && email.isNotBlank() && password.isNotBlank()
}

class FormViewModel : ViewModel() {
    private val _form = MutableStateFlow(FormState())
    val form: StateFlow<FormState> = _form.asStateFlow()

    fun updateName(name: String) {
        _form.update {
            it.copy(
                name = name,
                nameError = when {
                    name.isBlank() -> "名前を入力してください"
                    name.length < 2 -> "2文字以上で入力してください"
                    else -> null
                }
            )
        }
    }

    fun updateEmail(email: String) {
        _form.update {
            it.copy(
                email = email,
                emailError = when {
                    email.isBlank() -> "メールアドレスを入力してください"
                    !email.matches(Regex(".+@.+\\..+")) -> "正しい形式で入力してください"
                    else -> null
                }
            )
        }
    }

    fun updatePassword(password: String) {
        _form.update {
            it.copy(
                password = password,
                passwordError = when {
                    password.isBlank() -> "パスワードを入力してください"
                    password.length < 8 -> "8文字以上で入力してください"
                    !password.any { it.isUpperCase() } -> "大文字を含めてください"
                    !password.any { it.isDigit() } -> "数字を含めてください"
                    else -> null
                }
            )
        }
    }
}
```

---

## フォーム画面

```kotlin
@Composable
fun RegisterForm(viewModel: FormViewModel = viewModel()) {
    val form by viewModel.form.collectAsStateWithLifecycle()

    Column(Modifier.padding(16.dp)) {
        ValidatedField(
            value = form.name,
            onValueChange = { viewModel.updateName(it) },
            label = "名前",
            error = form.nameError
        )

        Spacer(Modifier.height(8.dp))

        ValidatedField(
            value = form.email,
            onValueChange = { viewModel.updateEmail(it) },
            label = "メールアドレス",
            error = form.emailError,
            keyboardType = KeyboardType.Email
        )

        Spacer(Modifier.height(8.dp))

        ValidatedField(
            value = form.password,
            onValueChange = { viewModel.updatePassword(it) },
            label = "パスワード",
            error = form.passwordError,
            keyboardType = KeyboardType.Password,
            isPassword = true
        )

        Spacer(Modifier.height(16.dp))

        Button(
            onClick = { /* 登録処理 */ },
            enabled = form.isValid,
            modifier = Modifier.fillMaxWidth()
        ) {
            Text("登録")
        }
    }
}

@Composable
fun ValidatedField(
    value: String,
    onValueChange: (String) -> Unit,
    label: String,
    error: String?,
    keyboardType: KeyboardType = KeyboardType.Text,
    isPassword: Boolean = false
) {
    OutlinedTextField(
        value = value,
        onValueChange = onValueChange,
        label = { Text(label) },
        isError = error != null,
        supportingText = error?.let { { Text(it) } },
        visualTransformation = if (isPassword) PasswordVisualTransformation() else VisualTransformation.None,
        keyboardOptions = KeyboardOptions(keyboardType = keyboardType),
        modifier = Modifier.fillMaxWidth()
    )
}
```

---

## デバウンス付きバリデーション

```kotlin
class SearchViewModel : ViewModel() {
    private val _query = MutableStateFlow("")
    val query: StateFlow<String> = _query.asStateFlow()

    // 300ms後にバリデーション実行
    val queryError: StateFlow<String?> = _query
        .debounce(300)
        .map { query ->
            when {
                query.isNotBlank() && query.length < 2 -> "2文字以上で検索してください"
                query.length > 100 -> "100文字以内で入力してください"
                else -> null
            }
        }
        .stateIn(viewModelScope, SharingStarted.Lazily, null)

    fun updateQuery(query: String) {
        _query.value = query
    }
}
```

---

## まとめ

- `OutlinedTextField`の`isError`/`supportingText`でエラー表示
- `ValidationResult`でバリデーション結果を構造化
- `FormState`データクラスでフォーム全体の状態管理
- `isValid`プロパティで送信ボタンのenable制御
- `debounce`で入力中の頻繁なバリデーションを抑制
- 共通の`ValidatedField`コンポーネントで再利用

---

8種類のAndroidアプリテンプレート（入力バリデーション設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [入力フォーム完全ガイド](https://zenn.dev/myougatheaxo/articles/compose-forms-input-2026)
- [フォーカス管理ガイド](https://zenn.dev/myougatheaxo/articles/android-compose-focus-management-2026)
- [パスワード強度チェッカー](https://zenn.dev/myougatheaxo/articles/android-compose-password-strength-2026)
