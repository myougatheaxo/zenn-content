---
title: "DI（依存性注入）の選び方 — 個人アプリにHiltは必要か？"
emoji: "💉"
type: "tech"
topics: ["android", "kotlin", "hilt", "architecture"]
published: true
---

## この記事で学べること

Androidの設計パターンを調べると「Hiltを使いましょう」という記事が大量に出てきます。でも、AIが生成するシンプルなアプリに本当にHiltが必要でしょうか？

**結論：個人アプリの80%はHilt不要**です。

---

## そもそもDIとは

DI（Dependency Injection）= **必要なものを外から渡す**こと。

```kotlin
// DIなし（ハードコーディング）
class HabitViewModel : ViewModel() {
    private val dao = AppDatabase.getInstance().habitDao() // ❌ 直接生成
}

// DIあり（外から渡す）
class HabitViewModel(
    private val dao: HabitDao // ✅ 外から受け取る
) : ViewModel()
```

DIのメリット：
- テスト時にFakeを渡せる
- 実装の差し替えが簡単
- 依存関係が明確になる

---

## 3つの選択肢

### 1. 手動DI（Manual Injection）

```kotlin
// Application.kt
class MyApp : Application() {
    val database by lazy { AppDatabase.getInstance(this) }
    val habitDao by lazy { database.habitDao() }
}

// ViewModelFactory
class HabitViewModelFactory(
    private val dao: HabitDao
) : ViewModelProvider.Factory {
    override fun <T : ViewModel> create(modelClass: Class<T>): T {
        return HabitViewModel(dao) as T
    }
}

// Composableでの使用
@Composable
fun HabitScreen() {
    val app = LocalContext.current.applicationContext as MyApp
    val viewModel: HabitViewModel = viewModel(
        factory = HabitViewModelFactory(app.habitDao)
    )
    // ...
}
```

**メリット**: フレームワーク不要、理解しやすい、ビルド速度に影響なし
**デメリット**: ボイラープレートが増える（大規模アプリでは）

### 2. Hilt

```kotlin
@HiltAndroidApp
class MyApp : Application()

@Module
@InstallIn(SingletonComponent::class)
object DatabaseModule {
    @Provides
    @Singleton
    fun provideDatabase(@ApplicationContext context: Context): AppDatabase {
        return AppDatabase.getInstance(context)
    }

    @Provides
    fun provideHabitDao(database: AppDatabase): HabitDao {
        return database.habitDao()
    }
}

@HiltViewModel
class HabitViewModel @Inject constructor(
    private val dao: HabitDao
) : ViewModel()
```

**メリット**: 大規模アプリで依存関係が自動解決される
**デメリット**: アノテーション処理でビルド時間増、学習コスト高い

### 3. Koin

```kotlin
val appModule = module {
    single { AppDatabase.getInstance(androidContext()) }
    single { get<AppDatabase>().habitDao() }
    viewModel { HabitViewModel(get()) }
}

// Application
class MyApp : Application() {
    override fun onCreate() {
        super.onCreate()
        startKoin {
            androidContext(this@MyApp)
            modules(appModule)
        }
    }
}
```

**メリット**: DSLベースで記述が簡潔、アノテーション処理なし
**デメリット**: 実行時エラー（コンパイル時に検出できない）

---

## どれを選ぶべきか

| 条件 | 推奨 |
|------|------|
| 画面5個以下 | 手動DI |
| 画面5〜15個 | Koin |
| 画面15個以上 | Hilt |
| テンプレート販売用 | 手動DI（依存フレームワーク最小化） |

AIが生成するアプリは画面2〜5個がほとんど。**手動DIで十分**です。

---

## AIの判断は正しい

Claude Codeで生成した8つのアプリを分析：

| アプリ | 画面数 | DI方式 |
|--------|--------|--------|
| ハビットトラッカー | 3 | 手動 |
| 家計簿 | 4 | 手動 |
| タスク管理 | 3 | 手動 |
| 予算管理 | 5 | 手動 |
| 割り勘計算 | 2 | 手動 |
| ミーティングタイマー | 2 | 手動 |
| 単位変換 | 2 | 手動 |
| カウントダウン | 3 | 手動 |

全て手動DI。そして全て正しい判断です。

Hiltを入れると：
- KSP/KAPTの設定が必要
- ビルド時間が30〜60秒増加
- `@HiltAndroidApp`、`@AndroidEntryPoint`、`@HiltViewModel`、`@Inject`のアノテーション学習が必要

画面3つのアプリにこのオーバーヘッドは不要です。

---

## まとめ

- DIの本質は「必要なものを外から渡す」こと
- **個人アプリの80%は手動DIで十分**
- Hiltはアプリが大きくなってから導入すればいい
- AIは規模に応じて適切なDI方式を選んでくれる

---

8種類のAndroidアプリテンプレート（シンプルな手動DI設計、フレームワーク依存なし）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [ViewModel + StateFlow入門](https://zenn.dev/myougatheaxo/articles/android-viewmodel-stateflow-2026)
- [AIが生成したアプリのテスト戦略](https://zenn.dev/myougatheaxo/articles/android-testing-ai-2026)
- [CLAUDE.mdの書き方完全ガイド](https://zenn.dev/myougatheaxo/articles/claude-md-best-practices-2026)
- [Google Play公開チェックリスト](https://zenn.dev/myougatheaxo/articles/google-play-publish-guide-2026)
