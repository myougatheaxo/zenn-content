---
title: "Compose PlayIntegrity完全ガイド — 改ざん検出/デバイス認証/不正防止"
emoji: "🛡️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "security"]
published: true
---

## この記事で学べること

**Compose PlayIntegrity**（Play Integrity API、デバイス認証、改ざん検出、サーバー検証）を解説します。

---

## セットアップ

```groovy
// build.gradle
dependencies {
    implementation("com.google.android.play:integrity:1.3.0")
}
```

---

## Integrity Token取得

```kotlin
class IntegrityChecker(private val context: Context) {
    suspend fun getIntegrityToken(nonce: String): String? {
        return try {
            val manager = IntegrityManagerFactory.create(context)
            val request = IntegrityTokenRequest.builder()
                .setNonce(nonce)
                .build()
            val response = manager.requestIntegrityToken(request).await()
            response.token()
        } catch (e: Exception) {
            null
        }
    }
}

// サーバーサイドで検証
// POST /api/verify-integrity
// Body: { "token": "<integrity_token>" }
// Google Play Integrity APIでトークンをデコード・検証
```

---

## Compose UI

```kotlin
@HiltViewModel
class SecurityViewModel @Inject constructor(
    private val integrityChecker: IntegrityChecker,
    private val api: ApiService
) : ViewModel() {

    private val _integrityState = MutableStateFlow<IntegrityState>(IntegrityState.Unchecked)
    val integrityState: StateFlow<IntegrityState> = _integrityState

    fun checkIntegrity() {
        viewModelScope.launch {
            _integrityState.value = IntegrityState.Checking
            val nonce = generateNonce()
            val token = integrityChecker.getIntegrityToken(nonce)
            if (token != null) {
                val result = api.verifyIntegrity(token)
                _integrityState.value = if (result.isValid) {
                    IntegrityState.Valid
                } else {
                    IntegrityState.Invalid(result.reason)
                }
            } else {
                _integrityState.value = IntegrityState.Error("トークン取得失敗")
            }
        }
    }

    private fun generateNonce(): String =
        Base64.encodeToString(UUID.randomUUID().toString().toByteArray(), Base64.NO_WRAP)
}

sealed class IntegrityState {
    data object Unchecked : IntegrityState()
    data object Checking : IntegrityState()
    data object Valid : IntegrityState()
    data class Invalid(val reason: String) : IntegrityState()
    data class Error(val message: String) : IntegrityState()
}

@Composable
fun IntegrityScreen(viewModel: SecurityViewModel = hiltViewModel()) {
    val state by viewModel.integrityState.collectAsStateWithLifecycle()

    Column(Modifier.padding(16.dp), horizontalAlignment = Alignment.CenterHorizontally,
        verticalArrangement = Arrangement.spacedBy(16.dp)) {
        when (state) {
            IntegrityState.Unchecked -> {
                Button(onClick = { viewModel.checkIntegrity() }) { Text("整合性チェック") }
            }
            IntegrityState.Checking -> CircularProgressIndicator()
            IntegrityState.Valid -> {
                Icon(Icons.Default.CheckCircle, null, Modifier.size(64.dp), tint = Color(0xFF4CAF50))
                Text("デバイスは安全です", style = MaterialTheme.typography.titleMedium)
            }
            is IntegrityState.Invalid -> {
                Icon(Icons.Default.Warning, null, Modifier.size(64.dp), tint = Color.Red)
                Text("問題が検出されました", style = MaterialTheme.typography.titleMedium)
            }
            is IntegrityState.Error -> {
                Text("エラー: ${(state as IntegrityState.Error).message}")
            }
        }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `IntegrityManager` | Integrityチェック |
| `IntegrityTokenRequest` | トークンリクエスト |
| `nonce` | リプレイ攻撃防止 |
| サーバー検証 | トークン復号・判定 |

- Play Integrity APIでデバイスの改ざん・Root検出
- nonceでリプレイ攻撃を防止
- トークンはサーバーサイドで検証（クライアントでは復号不可）
- 課金処理や機密データアクセス前にチェック推奨

---

8種類のAndroidアプリテンプレート（セキュリティ対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose Biometric](https://zenn.dev/myougatheaxo/articles/android-compose-compose-biometric-2026)
- [Compose CredentialManager](https://zenn.dev/myougatheaxo/articles/android-compose-compose-credential-manager-2026)
- [Compose InAppPurchase](https://zenn.dev/myougatheaxo/articles/android-compose-compose-in-app-purchase-2026)
