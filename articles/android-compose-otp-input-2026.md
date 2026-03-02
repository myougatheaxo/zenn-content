---
title: "OTP入力完全ガイド — ワンタイムパスワード/自動入力/SMS連携"
emoji: "🔑"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "auth"]
published: true
---

## この記事で学べること

**OTP入力**（ワンタイムパスワードUI、SMS自動読取、各桁独立入力、フォーカス制御）を解説します。

---

## OTP入力コンポーネント

```kotlin
@Composable
fun OtpInput(
    length: Int = 6,
    onComplete: (String) -> Unit
) {
    var otp by remember { mutableStateOf(List(length) { "" }) }
    val focusRequesters = remember { List(length) { FocusRequester() } }

    LaunchedEffect(Unit) { focusRequesters[0].requestFocus() }

    Row(
        Modifier.padding(16.dp),
        horizontalArrangement = Arrangement.spacedBy(8.dp)
    ) {
        repeat(length) { index ->
            OutlinedTextField(
                value = otp[index],
                onValueChange = { value ->
                    if (value.length <= 1 && value.all { it.isDigit() }) {
                        otp = otp.toMutableList().apply { this[index] = value }

                        if (value.isNotEmpty() && index < length - 1) {
                            focusRequesters[index + 1].requestFocus()
                        }

                        val code = otp.joinToString("")
                        if (code.length == length) onComplete(code)
                    }
                },
                modifier = Modifier
                    .width(48.dp)
                    .focusRequester(focusRequesters[index]),
                textStyle = TextStyle(
                    fontSize = 24.sp,
                    textAlign = TextAlign.Center,
                    fontWeight = FontWeight.Bold
                ),
                keyboardOptions = KeyboardOptions(keyboardType = KeyboardType.Number),
                singleLine = true
            )
        }
    }
}
```

---

## カスタムデザインOTP

```kotlin
@Composable
fun StyledOtpInput(
    length: Int = 6,
    value: String,
    onValueChange: (String) -> Unit
) {
    val focusRequester = remember { FocusRequester() }

    LaunchedEffect(Unit) { focusRequester.requestFocus() }

    // 隠しTextField
    BasicTextField(
        value = value,
        onValueChange = { if (it.length <= length && it.all { c -> c.isDigit() }) onValueChange(it) },
        keyboardOptions = KeyboardOptions(keyboardType = KeyboardType.Number),
        modifier = Modifier.size(0.dp).focusRequester(focusRequester)
    )

    // 表示用ボックス
    Row(
        Modifier
            .clickable { focusRequester.requestFocus() }
            .padding(16.dp),
        horizontalArrangement = Arrangement.spacedBy(12.dp)
    ) {
        repeat(length) { index ->
            val char = value.getOrNull(index)?.toString() ?: ""
            val isFocused = value.length == index

            Box(
                Modifier
                    .size(50.dp)
                    .border(
                        width = if (isFocused) 2.dp else 1.dp,
                        color = if (isFocused) MaterialTheme.colorScheme.primary
                        else MaterialTheme.colorScheme.outline,
                        shape = RoundedCornerShape(12.dp)
                    ),
                contentAlignment = Alignment.Center
            ) {
                Text(
                    text = char,
                    fontSize = 24.sp,
                    fontWeight = FontWeight.Bold
                )

                // カーソル点滅
                if (isFocused) {
                    val alpha by rememberInfiniteTransition(label = "cursor")
                        .animateFloat(
                            initialValue = 0f, targetValue = 1f,
                            animationSpec = infiniteRepeatable(tween(500), RepeatMode.Reverse),
                            label = "cursorAlpha"
                        )
                    Box(
                        Modifier
                            .width(2.dp)
                            .height(24.dp)
                            .background(MaterialTheme.colorScheme.primary.copy(alpha = alpha))
                    )
                }
            }
        }
    }
}
```

---

## SMS自動読取

```kotlin
// Google SMS User Consent API
class SmsVerificationActivity : ComponentActivity() {
    private val smsLauncher = registerForActivityResult(
        ActivityResultContracts.StartActivityForResult()
    ) { result ->
        if (result.resultCode == RESULT_OK) {
            val message = result.data?.getStringExtra(SmsRetriever.EXTRA_SMS_MESSAGE) ?: ""
            val otp = Regex("\\d{6}").find(message)?.value ?: ""
            // OTPを自動入力
        }
    }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        val client = SmsRetriever.getClient(this)
        client.startSmsUserConsent(null)

        registerReceiver(object : BroadcastReceiver() {
            override fun onReceive(context: Context, intent: Intent) {
                if (SmsRetriever.SMS_RETRIEVED_ACTION == intent.action) {
                    val consentIntent = intent.getParcelableExtra<Intent>(SmsRetriever.EXTRA_CONSENT_INTENT)
                    consentIntent?.let { smsLauncher.launch(it) }
                }
            }
        }, IntentFilter(SmsRetriever.SMS_RETRIEVED_ACTION), RECEIVER_NOT_EXPORTED)
    }
}
```

---

## まとめ

| 機能 | 実装 |
|------|------|
| 各桁入力 | `FocusRequester`制御 |
| カスタムUI | 隠し`BasicTextField` |
| SMS読取 | `SmsRetriever` API |
| カーソル | `infiniteRepeatable`アニメーション |

- `FocusRequester`で入力後に自動フォーカス移動
- 隠し`BasicTextField`でカスタムUI実現
- SMS User Consent APIで6桁コード自動入力
- カーソル点滅アニメーションでUX向上

---

8種類のAndroidアプリテンプレート（認証UI実装済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [数字パッド](https://zenn.dev/myougatheaxo/articles/android-compose-number-pad-2026)
- [フォームバリデーション](https://zenn.dev/myougatheaxo/articles/android-compose-form-validation-2026)
- [Credential Manager](https://zenn.dev/myougatheaxo/articles/android-compose-credential-manager-2026)
