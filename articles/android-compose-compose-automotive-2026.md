---
title: "Compose Automotive完全ガイド — Android Auto/車載アプリ/CarAppService"
emoji: "🚗"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "automotive"]
published: true
---

## この記事で学べること

**Compose Automotive**（Android Auto、CarAppService、車載UI、テンプレートパターン）を解説します。

---

## セットアップ

```groovy
// build.gradle
dependencies {
    implementation("androidx.car.app:app:1.4.0")
}
```

---

## CarAppService

```kotlin
class MyCarAppService : CarAppService() {
    override fun createHostValidator(): HostValidator =
        HostValidator.ALLOW_ALL_HOSTS_VALIDATOR

    override fun onCreateSession(): Session = MyCarSession()
}

class MyCarSession : Session() {
    override fun onCreateScreen(intent: Intent): Screen = MainScreen(carContext)
}

class MainScreen(carContext: CarContext) : Screen(carContext) {
    override fun onGetTemplate(): Template {
        val listBuilder = ItemList.Builder()

        listBuilder.addItem(
            Row.Builder()
                .setTitle("ナビ開始")
                .setImage(CarIcon.Builder(IconCompat.createWithResource(carContext,
                    R.drawable.ic_navigation)).build(), Row.IMAGE_TYPE_ICON)
                .setOnClickListener { /* ナビ開始 */ }
                .build()
        )

        listBuilder.addItem(
            Row.Builder()
                .setTitle("お気に入り")
                .addText("登録済み: 5件")
                .setBrowsable(true)
                .setOnClickListener {
                    screenManager.push(FavoritesScreen(carContext))
                }
                .build()
        )

        return ListTemplate.Builder()
            .setTitle("マイカーアプリ")
            .setSingleList(listBuilder.build())
            .setHeaderAction(Action.APP_ICON)
            .build()
    }
}
```

---

## AndroidManifest

```xml
<application>
    <service android:name=".MyCarAppService"
        android:exported="true">
        <intent-filter>
            <action android:name="androidx.car.app.CarAppService" />
            <category android:name="androidx.car.app.category.NAVIGATION" />
        </intent-filter>
    </service>

    <meta-data android:name="androidx.car.app.minCarApiLevel" android:value="1" />
</application>
```

---

## ナビゲーションテンプレート

```kotlin
class NavigationScreen(carContext: CarContext) : Screen(carContext) {
    override fun onGetTemplate(): Template {
        val actionStrip = ActionStrip.Builder()
            .addAction(Action.Builder()
                .setIcon(CarIcon.Builder(IconCompat.createWithResource(carContext,
                    R.drawable.ic_search)).build())
                .setOnClickListener { /* 検索 */ }
                .build())
            .build()

        return NavigationTemplate.Builder()
            .setActionStrip(actionStrip)
            .setDestinationTravelEstimate(
                TravelEstimate.Builder(
                    Distance.create(15.5, Distance.UNIT_KILOMETERS),
                    DateTimeWithZone.create(System.currentTimeMillis() + 1200000,
                        TimeZone.getDefault())
                ).setRemainingTimeSeconds(1200).build()
            )
            .build()
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `CarAppService` | 車載アプリサービス |
| `ListTemplate` | リスト画面 |
| `NavigationTemplate` | ナビ画面 |
| `ScreenManager` | 画面遷移 |

- Android Autoはテンプレートベースのアプリ開発
- `CarAppService`でアプリのエントリーポイントを定義
- `screenManager.push()`で画面遷移
- 運転中の安全のためUIはテンプレートに制限される

---

8種類のAndroidアプリテンプレート（Material3対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose TV](https://zenn.dev/myougatheaxo/articles/android-compose-compose-tv-2026)
- [Compose WearOS](https://zenn.dev/myougatheaxo/articles/android-compose-compose-wear-os-2026)
- [Compose Location](https://zenn.dev/myougatheaxo/articles/android-compose-compose-location-2026)
