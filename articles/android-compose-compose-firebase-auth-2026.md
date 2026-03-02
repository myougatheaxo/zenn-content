---
title: "Compose Firebase Auth完全ガイド — メール認証/Google認証/状態管理"
emoji: "🔐"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "firebase"]
published: true
---

## この記事で学べること

**Compose Firebase Auth**（メール/パスワード認証、Googleサインイン、認証状態管理）を解説します。

---

## メール/パスワード認証

```kotlin
@HiltViewModel
class AuthViewModel @Inject constructor() : ViewModel() {
    private val auth = Firebase.auth

    private val _authState = MutableStateFlow<AuthState>(AuthState.Loading)
    val authState: StateFlow<AuthState> = _authState.asStateFlow()

    init {
        auth.addAuthStateListener { firebaseAuth ->
            _authState.value = firebaseAuth.currentUser?.let { AuthState.Authenticated(it) }
                ?: AuthState.Unauthenticated
        }
    }

    fun signIn(email: String, password: String) {
        viewModelScope.launch {
            _authState.value = AuthState.Loading
            auth.signInWithEmailAndPassword(email, password)
                .addOnFailureListener { _authState.value = AuthState.Error(it.message ?: "エラー") }
        }
    }

    fun signUp(email: String, password: String) {
        viewModelScope.launch {
            _authState.value = AuthState.Loading
            auth.createUserWithEmailAndPassword(email, password)
                .addOnFailureListener { _authState.value = AuthState.Error(it.message ?: "エラー") }
        }
    }

    fun signOut() { auth.signOut() }
}

sealed class AuthState {
    data object Loading : AuthState()
    data object Unauthenticated : AuthState()
    data class Authenticated(val user: FirebaseUser) : AuthState()
    data class Error(val message: String) : AuthState()
}
```

---

## ログインUI

```kotlin
@Composable
fun LoginScreen(viewModel: AuthViewModel = hiltViewModel(), onLoggedIn: () -> Unit) {
    val authState by viewModel.authState.collectAsStateWithLifecycle()
    var email by rememberSaveable { mutableStateOf("") }
    var password by rememberSaveable { mutableStateOf("") }

    LaunchedEffect(authState) {
        if (authState is AuthState.Authenticated) onLoggedIn()
    }

    Column(Modifier.fillMaxSize().padding(16.dp), verticalArrangement = Arrangement.Center) {
        OutlinedTextField(value = email, onValueChange = { email = it }, label = { Text("メール") },
            modifier = Modifier.fillMaxWidth(), keyboardOptions = KeyboardOptions(keyboardType = KeyboardType.Email))
        Spacer(Modifier.height(8.dp))
        OutlinedTextField(value = password, onValueChange = { password = it }, label = { Text("パスワード") },
            modifier = Modifier.fillMaxWidth(), visualTransformation = PasswordVisualTransformation())
        Spacer(Modifier.height(16.dp))

        Button(onClick = { viewModel.signIn(email, password) }, Modifier.fillMaxWidth(),
            enabled = authState !is AuthState.Loading) { Text("ログイン") }
        OutlinedButton(onClick = { viewModel.signUp(email, password) }, Modifier.fillMaxWidth()) { Text("新規登録") }

        if (authState is AuthState.Error) {
            Text((authState as AuthState.Error).message, color = MaterialTheme.colorScheme.error,
                modifier = Modifier.padding(top = 8.dp))
        }
    }
}
```

---

## Googleサインイン

```kotlin
@Composable
fun GoogleSignInButton(viewModel: AuthViewModel = hiltViewModel()) {
    val context = LocalContext.current
    val launcher = rememberLauncherForActivityResult(ActivityResultContracts.StartIntentSenderForResult()) { result ->
        val credential = Identity.getSignInClient(context).getSignInCredentialFromIntent(result.data)
        val idToken = credential.googleIdToken
        if (idToken != null) {
            val firebaseCredential = GoogleAuthProvider.getCredential(idToken, null)
            Firebase.auth.signInWithCredential(firebaseCredential)
        }
    }

    OutlinedButton(onClick = {
        val signInRequest = BeginSignInRequest.builder()
            .setGoogleIdTokenRequestOptions(
                BeginSignInRequest.GoogleIdTokenRequestOptions.builder()
                    .setSupported(true)
                    .setServerClientId("YOUR_WEB_CLIENT_ID")
                    .setFilterByAuthorizedAccounts(false)
                    .build()
            ).build()

        Identity.getSignInClient(context).beginSignIn(signInRequest)
            .addOnSuccessListener { result ->
                launcher.launch(IntentSenderRequest.Builder(result.pendingIntent.intentSender).build())
            }
    }, Modifier.fillMaxWidth()) {
        Text("Googleでサインイン")
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `Firebase.auth` | 認証インスタンス |
| `signInWithEmailAndPassword` | メール認証 |
| `signInWithCredential` | Google認証 |
| `addAuthStateListener` | 認証状態監視 |

- `AuthStateListener`で認証状態をリアクティブに監視
- `sealed class`で状態管理を型安全に
- Googleサインインは`Identity API`(新API)推奨
- `rememberSaveable`でフォーム状態をプロセス終了対応

---

8種類のAndroidアプリテンプレート（Firebase認証対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Firebase Firestore](https://zenn.dev/myougatheaxo/articles/android-compose-compose-firebase-firestore-2026)
- [Firebase Messaging](https://zenn.dev/myougatheaxo/articles/android-compose-compose-firebase-messaging-2026)
- [Compose Biometric](https://zenn.dev/myougatheaxo/articles/android-compose-compose-biometric-2026)
