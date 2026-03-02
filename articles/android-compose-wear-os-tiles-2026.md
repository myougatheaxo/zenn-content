---
title: "Wear OS Tiles完全ガイド — タイルAPI/レイアウト/データ更新"
emoji: "⌚"
type: "tech"
topics: ["android", "kotlin", "wearos", "jetpackcompose"]
published: true
---

## この記事で学べること

**Wear OS Tiles**（TileService、レイアウト構築、データ更新、クリックアクション、プレビュー）を解説します。

---

## セットアップ

```kotlin
dependencies {
    implementation("androidx.wear.tiles:tiles:1.4.1")
    implementation("androidx.wear.tiles:tiles-material3:1.4.1")
    implementation("androidx.wear.tiles:tiles-tooling-preview:1.4.1")
    debugImplementation("androidx.wear.tiles:tiles-tooling:1.4.1")
}
```

---

## TileService

```kotlin
class StepsTileService : TileService() {
    override suspend fun tileRequest(requestParams: RequestBuilders.TileRequest): TileBuilders.Tile {
        val steps = getStepsFromDataStore()

        return TileBuilders.Tile.Builder()
            .setResourcesVersion("1")
            .setTileTimeline(
                TimelineBuilders.Timeline.Builder()
                    .addTimelineEntry(
                        TimelineBuilders.TimelineEntry.Builder()
                            .setLayout(
                                LayoutElementBuilders.Layout.Builder()
                                    .setRoot(createLayout(steps))
                                    .build()
                            )
                            .build()
                    )
                    .build()
            )
            .setFreshnessIntervalMillis(300_000) // 5分ごとに更新
            .build()
    }

    override suspend fun resourcesRequest(
        requestParams: RequestBuilders.ResourcesRequest
    ): ResourceBuilders.Resources {
        return ResourceBuilders.Resources.Builder()
            .setVersion("1")
            .addIdToImageMapping("icon_walk", ResourceBuilders.ImageResource.Builder()
                .setAndroidResourceByResId(
                    ResourceBuilders.AndroidImageResourceByResId.Builder()
                        .setResourceId(R.drawable.ic_walk)
                        .build()
                ).build()
            )
            .build()
    }

    private fun createLayout(steps: Int): LayoutElementBuilders.LayoutElement {
        return LayoutElementBuilders.Column.Builder()
            .addContent(
                LayoutElementBuilders.Image.Builder()
                    .setResourceId("icon_walk")
                    .setWidth(DimensionBuilders.dp(24f))
                    .setHeight(DimensionBuilders.dp(24f))
                    .build()
            )
            .addContent(
                LayoutElementBuilders.Text.Builder()
                    .setText("$steps")
                    .setFontStyle(
                        LayoutElementBuilders.FontStyle.Builder()
                            .setSize(DimensionBuilders.sp(32f))
                            .setWeight(LayoutElementBuilders.FONT_WEIGHT_BOLD)
                            .build()
                    )
                    .build()
            )
            .addContent(
                LayoutElementBuilders.Text.Builder()
                    .setText("歩")
                    .setFontStyle(
                        LayoutElementBuilders.FontStyle.Builder()
                            .setSize(DimensionBuilders.sp(14f))
                            .build()
                    )
                    .build()
            )
            .setHorizontalAlignment(LayoutElementBuilders.HORIZONTAL_ALIGN_CENTER)
            .build()
    }
}
```

---

## マニフェスト登録

```xml
<service
    android:name=".StepsTileService"
    android:exported="true"
    android:permission="com.google.android.wearable.permission.BIND_TILE_PROVIDER">
    <intent-filter>
        <action android:name="androidx.wear.tiles.action.BIND_TILE_PROVIDER" />
    </intent-filter>
    <meta-data
        android:name="androidx.wear.tiles.PREVIEW"
        android:resource="@drawable/tile_preview" />
</service>
```

---

## クリックアクション

```kotlin
private fun createClickableLayout(): LayoutElementBuilders.LayoutElement {
    val clickable = ModifiersBuilders.Clickable.Builder()
        .setOnClick(
            ActionBuilders.LaunchAction.Builder()
                .setAndroidActivity(
                    ActionBuilders.AndroidActivity.Builder()
                        .setClassName("com.example.MainActivity")
                        .setPackageName("com.example.app")
                        .build()
                )
                .build()
        )
        .setId("open_app")
        .build()

    return LayoutElementBuilders.Box.Builder()
        .addContent(/* content */)
        .setModifiers(
            ModifiersBuilders.Modifiers.Builder()
                .setClickable(clickable)
                .build()
        )
        .build()
}
```

---

## タイル更新トリガー

```kotlin
// データ変更時にタイル更新を要求
class StepsRepository @Inject constructor(context: Context) {
    fun updateSteps(steps: Int) {
        // データ保存
        saveSteps(steps)
        // タイル更新要求
        TileService.getUpdater(context).requestUpdate(StepsTileService::class.java)
    }
}
```

---

## まとめ

| 機能 | 実装 |
|------|------|
| タイル定義 | `TileService` |
| レイアウト | `LayoutElementBuilders` |
| リソース | `ResourceBuilders` |
| クリック | `LaunchAction` |
| 更新 | `TileService.getUpdater()` |

- `TileService`でWear OSタイルを定義
- `LayoutElementBuilders`でUIを構築
- `freshnessIntervalMillis`で定期更新
- `requestUpdate()`で即座にタイル更新

---

8種類のAndroidアプリテンプレート（Wear OS対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Wear OS基本](https://zenn.dev/myougatheaxo/articles/android-compose-wear-os-basics-2026)
- [App Widget Glance](https://zenn.dev/myougatheaxo/articles/android-compose-app-widget-glance-2026)
- [Health Connect](https://zenn.dev/myougatheaxo/articles/android-compose-health-connect-2026)
