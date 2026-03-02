---
title: "Hilt CustomScope完全ガイド — カスタムスコープ/Activity/Fragment/ViewModel"
emoji: "🎯"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "hilt"]
published: true
---

## この記事で学べること

**Hilt CustomScope**（スコープ使い分け、ActivityScoped、ViewModelScoped、カスタムEntryPoint）を解説します。

---

## Hiltスコープ一覧

```kotlin
// スコープの階層（上ほど長生き）
// SingletonComponent   → @Singleton          → アプリ全体
// ActivityRetainedComponent → @ActivityRetainedScoped → 構成変更をまたぐ
// ViewModelComponent   → @ViewModelScoped    → ViewModel単位
// ActivityComponent    → @ActivityScoped     → Activity単位
// FragmentComponent    → @FragmentScoped     → Fragment単位
// ViewComponent        → @ViewScoped         → View単位
// ServiceComponent     → @ServiceScoped      → Service単位

// Singleton: アプリ全体で1インスタンス
@Module
@InstallIn(SingletonComponent::class)
object AppModule {
    @Provides @Singleton
    fun provideDatabase(@ApplicationContext ctx: Context): AppDatabase =
        Room.databaseBuilder(ctx, AppDatabase::class.java, "app.db").build()
}

// ActivityRetainedScoped: 画面回転でも保持
@Module
@InstallIn(ActivityRetainedComponent::class)
object ActivityRetainedModule {
    @Provides @ActivityRetainedScoped
    fun provideSessionManager(): SessionManager = SessionManager()
}
```

---

## ViewModelScoped

```kotlin
@Module
@InstallIn(ViewModelComponent::class)
object ViewModelModule {
    @Provides @ViewModelScoped
    fun provideFormValidator(): FormValidator = FormValidator()
}

// ViewModelで自動注入
@HiltViewModel
class RegistrationViewModel @Inject constructor(
    private val validator: FormValidator,  // ViewModel単位でスコープ
    private val repository: UserRepository // Singletonスコープ
) : ViewModel() {
    fun validate(email: String) = validator.validateEmail(email)
}

@Composable
fun RegistrationScreen(viewModel: RegistrationViewModel = hiltViewModel()) {
    var email by remember { mutableStateOf("") }
    var isValid by remember { mutableStateOf(true) }

    Column(Modifier.padding(16.dp)) {
        OutlinedTextField(
            value = email, onValueChange = {
                email = it; isValid = viewModel.validate(it)
            },
            label = { Text("メールアドレス") },
            isError = !isValid
        )
    }
}
```

---

## EntryPoint（Hilt外からのアクセス）

```kotlin
// ContentProviderなどHilt管理外からDIオブジェクトを取得
@EntryPoint
@InstallIn(SingletonComponent::class)
interface DatabaseEntryPoint {
    fun database(): AppDatabase
}

class MyContentProvider : ContentProvider() {
    override fun query(uri: Uri, projection: Array<String>?, selection: String?,
        selectionArgs: Array<String>?, sortOrder: String?): Cursor? {
        val entryPoint = EntryPointAccessors.fromApplication(
            context!!.applicationContext, DatabaseEntryPoint::class.java
        )
        val db = entryPoint.database()
        return db.userDao().getAllCursor()
    }
    // ... 他のメソッド
}
```

---

## まとめ

| スコープ | ライフタイム |
|----------|-------------|
| `@Singleton` | アプリ全体 |
| `@ActivityRetainedScoped` | 構成変更をまたぐ |
| `@ViewModelScoped` | ViewModel単位 |
| `@ActivityScoped` | Activity単位 |

- スコープは必要最小限に設定（メモリ効率）
- `@Singleton`は本当にアプリ全体で必要なものだけ
- `@ViewModelScoped`はフォーム検証等のViewModel固有ロジック
- `@EntryPoint`でHilt管理外からDIオブジェクトにアクセス

---

8種類のAndroidアプリテンプレート（Hilt DI対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Hilt Module](https://zenn.dev/myougatheaxo/articles/android-compose-hilt-module-2026)
- [Hilt ViewModel](https://zenn.dev/myougatheaxo/articles/android-compose-hilt-viewmodel-2026)
- [Hilt Multibinding](https://zenn.dev/myougatheaxo/articles/android-compose-hilt-multibinding-2026)
