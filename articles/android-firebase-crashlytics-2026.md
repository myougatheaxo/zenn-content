---
title: "Firebase Crashlytics導入ガイド — クラッシュを検知して品質を上げる"
emoji: "🔥"
type: "tech"
topics: ["android", "kotlin", "firebase", "crashlytics"]
published: true
---

## この記事で学べること

**Firebase Crashlytics**を導入すれば、ユーザーのクラッシュをリアルタイムで把握できます。セットアップから実践的な活用方法まで解説します。

---

## セットアップ

### 1. Firebase プロジェクト作成

Firebase Console（console.firebase.google.com）でプロジェクトを作成し、`google-services.json`をダウンロード。

### 2. 依存関係の追加

```kotlin
// プロジェクトレベル build.gradle.kts
plugins {
    id("com.google.gms.google-services") version "4.4.2" apply false
    id("com.google.firebase.crashlytics") version "3.0.2" apply false
}

// アプリレベル build.gradle.kts
plugins {
    id("com.google.gms.google-services")
    id("com.google.firebase.crashlytics")
}

dependencies {
    implementation(platform("com.google.firebase:firebase-bom:33.1.0"))
    implementation("com.google.firebase:firebase-crashlytics")
    implementation("com.google.firebase:firebase-analytics")
}
```

### 3. google-services.json を配置

```
app/
├── google-services.json  ← ここ
├── build.gradle.kts
└── src/
```

---

## 基本的な使い方

セットアップ後は**自動でクラッシュが収集**されます。追加コードは不要です。

### 手動でエラーを記録

```kotlin
try {
    riskyOperation()
} catch (e: Exception) {
    Firebase.crashlytics.recordException(e)
}
```

### カスタムログ

```kotlin
Firebase.crashlytics.log("ユーザーが設定画面を開いた")
Firebase.crashlytics.setCustomKey("screen", "settings")
Firebase.crashlytics.setUserId("user_123")
```

---

## Non-Fatalエラーの記録

```kotlin
class ApiRepository(private val api: MyApi) {
    suspend fun fetchData(): Result<Data> {
        return try {
            Result.success(api.getData())
        } catch (e: Exception) {
            // クラッシュはしないが、Crashlyticsに記録
            Firebase.crashlytics.recordException(e)
            Result.failure(e)
        }
    }
}
```

---

## ViewModelでの活用

```kotlin
class HomeViewModel(
    private val repository: DataRepository
) : ViewModel() {

    fun loadData() {
        viewModelScope.launch {
            try {
                val data = repository.getData()
                _uiState.value = UiState.Success(data)
            } catch (e: Exception) {
                Firebase.crashlytics.apply {
                    log("loadData failed")
                    setCustomKey("error_type", e.javaClass.simpleName)
                    recordException(e)
                }
                _uiState.value = UiState.Error(e.message)
            }
        }
    }
}
```

---

## デバッグビルドでの無効化

```kotlin
// Application クラス
class MyApp : Application() {
    override fun onCreate() {
        super.onCreate()
        Firebase.crashlytics.setCrashlyticsCollectionEnabled(!BuildConfig.DEBUG)
    }
}
```

開発中のクラッシュでダッシュボードが汚れるのを防止。

---

## テストクラッシュ

```kotlin
// 動作確認用（リリース前に削除すること）
Button(onClick = {
    throw RuntimeException("Crashlytics テストクラッシュ")
}) {
    Text("テストクラッシュ")
}
```

---

## Crashlyticsダッシュボードの見方

| 項目 | 内容 |
|------|------|
| Crash-free users | クラッシュなしのユーザー割合 |
| Issues | クラッシュの種類ごとのグループ |
| Events | 発生回数 |
| Users | 影響を受けたユーザー数 |
| Stack trace | クラッシュ箇所のコード |

**目標: Crash-free rate 99.5%以上**

---

## まとめ

- `firebase-crashlytics`を追加するだけで自動収集開始
- `recordException()`でNon-Fatalエラーも記録
- `setCustomKey()`/`log()`で追加情報を付与
- デバッグビルドでは無効化
- Crash-free rate 99.5%を目標に

---

8種類のAndroidアプリテンプレート（Crashlytics追加可能な設計）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [テスト入門（AI時代のAndroidテスト）](https://zenn.dev/myougatheaxo/articles/android-testing-ai-2026)
- [ProGuard/R8完全ガイド](https://zenn.dev/myougatheaxo/articles/android-proguard-r8-2026)
- [Google Play公開チェックリスト](https://zenn.dev/myougatheaxo/articles/google-play-publish-guide-2026)
