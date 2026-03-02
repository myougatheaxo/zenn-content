---
title: "Compose マルチモジュール完全ガイド — モジュール分割/依存関係/ビルド高速化"
emoji: "📦"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "architecture"]
published: true
---

## この記事で学べること

**Compose マルチモジュール**（モジュール分割戦略、依存関係設計、ビルド高速化、Hilt連携）を解説します。

---

## モジュール構成

```
app/                     # アプリモジュール（エントリーポイント）
├── feature/
│   ├── home/           # ホーム画面
│   ├── search/         # 検索画面
│   └── settings/       # 設定画面
├── core/
│   ├── ui/             # 共通Composeコンポーネント
│   ├── data/           # Repository/DataSource
│   ├── domain/         # UseCase/Entity
│   ├── network/        # Retrofit/API
│   ├── database/       # Room
│   └── common/         # ユーティリティ
```

```groovy
// settings.gradle.kts
include(":app")
include(":feature:home")
include(":feature:search")
include(":feature:settings")
include(":core:ui")
include(":core:data")
include(":core:domain")
include(":core:network")
include(":core:database")
include(":core:common")
```

---

## 依存関係

```groovy
// app/build.gradle.kts
dependencies {
    implementation(project(":feature:home"))
    implementation(project(":feature:search"))
    implementation(project(":feature:settings"))
}

// feature/home/build.gradle.kts
dependencies {
    implementation(project(":core:ui"))
    implementation(project(":core:domain"))
    // core:dataには直接依存しない（domain経由）
}

// core/domain/build.gradle.kts
dependencies {
    implementation(project(":core:common"))
    // dataへの依存なし（インターフェースのみ定義）
}

// core/data/build.gradle.kts
dependencies {
    implementation(project(":core:domain"))
    implementation(project(":core:network"))
    implementation(project(":core:database"))
}
```

---

## Navigation連携

```kotlin
// feature/home のルート
@Serializable object HomeRoute

// app/で統合
@Composable
fun AppNavHost() {
    val navController = rememberNavController()

    NavHost(navController, startDestination = HomeRoute) {
        homeNavGraph(navController)
        searchNavGraph(navController)
        settingsNavGraph(navController)
    }
}

// feature/home/navigation/HomeNavigation.kt
fun NavGraphBuilder.homeNavGraph(navController: NavController) {
    composable<HomeRoute> {
        HomeScreen(
            onSearchClick = { navController.navigate(SearchRoute) },
            onSettingsClick = { navController.navigate(SettingsRoute) }
        )
    }
}
```

---

## まとめ

| モジュール | 役割 |
|-----------|------|
| `app` | エントリー+DI統合 |
| `feature/*` | 画面単位 |
| `core/domain` | ビジネスロジック |
| `core/data` | データ層実装 |

- featureモジュールはcore/domainに依存（dataに直接依存しない）
- `api`/`implementation`で依存の公開範囲を制御
- Navigation拡張関数で各featureのルーティングを分離
- マルチモジュールでビルドの並列化・キャッシュが効く

---

8種類のAndroidアプリテンプレート（マルチモジュール対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose TypeSafeNavigation](https://zenn.dev/myougatheaxo/articles/android-compose-compose-type-safe-navigation-2026)
- [Hilt Module](https://zenn.dev/myougatheaxo/articles/android-compose-hilt-module-2026)
- [Gradle VersionCatalog](https://zenn.dev/myougatheaxo/articles/android-compose-gradle-version-catalog-2026)
