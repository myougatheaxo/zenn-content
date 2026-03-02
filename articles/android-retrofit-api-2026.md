---
title: "Retrofit入門 — AIが作ったAndroidアプリにAPI連携を追加する"
emoji: "🌐"
type: "tech"
topics: ["android", "kotlin", "retrofit", "api"]
published: true
---

## この記事で学べること

AIで生成したアプリは、ローカルデータ（Room Database）で完結しています。しかし「天気情報を取得したい」「為替レートを表示したい」と思ったら、**外部API**との連携が必要です。

AndroidでAPI通信をするなら、**Retrofit**一択です。

---

## Retrofitとは

Square社が開発したHTTPクライアントライブラリ。AndroidのAPI通信のデファクトスタンダードです。

```kotlin
// build.gradle.kts
dependencies {
    implementation("com.squareup.retrofit2:retrofit:2.11.0")
    implementation("com.squareup.retrofit2:converter-gson:2.11.0")
}
```

---

## 基本の3ステップ

### Step 1: APIインターフェース定義

```kotlin
interface WeatherApi {
    @GET("weather")
    suspend fun getWeather(
        @Query("q") city: String,
        @Query("appid") apiKey: String
    ): WeatherResponse
}

data class WeatherResponse(
    val main: Main,
    val name: String
)

data class Main(
    val temp: Double,
    val humidity: Int
)
```

### Step 2: Retrofitインスタンス作成

```kotlin
object RetrofitClient {
    private const val BASE_URL = "https://api.openweathermap.org/data/2.5/"

    val weatherApi: WeatherApi by lazy {
        Retrofit.Builder()
            .baseUrl(BASE_URL)
            .addConverterFactory(GsonConverterFactory.create())
            .build()
            .create(WeatherApi::class.java)
    }
}
```

### Step 3: ViewModelから呼び出し

```kotlin
class WeatherViewModel : ViewModel() {

    private val _weather = MutableStateFlow<WeatherResponse?>(null)
    val weather: StateFlow<WeatherResponse?> = _weather.asStateFlow()

    private val _error = MutableStateFlow<String?>(null)
    val error: StateFlow<String?> = _error.asStateFlow()

    fun fetchWeather(city: String) {
        viewModelScope.launch {
            try {
                val response = RetrofitClient.weatherApi.getWeather(
                    city = city,
                    apiKey = "YOUR_API_KEY"
                )
                _weather.value = response
            } catch (e: Exception) {
                _error.value = e.message
            }
        }
    }
}
```

---

## エラーハンドリング

API通信は失敗する前提で設計します。

```kotlin
sealed class ApiResult<out T> {
    data class Success<T>(val data: T) : ApiResult<T>()
    data class Error(val message: String) : ApiResult<Nothing>()
    object Loading : ApiResult<Nothing>()
}
```

```kotlin
private val _state = MutableStateFlow<ApiResult<WeatherResponse>>(ApiResult.Loading)
val state: StateFlow<ApiResult<WeatherResponse>> = _state.asStateFlow()

fun fetchWeather(city: String) {
    viewModelScope.launch {
        _state.value = ApiResult.Loading
        try {
            val response = RetrofitClient.weatherApi.getWeather(city, API_KEY)
            _state.value = ApiResult.Success(response)
        } catch (e: HttpException) {
            _state.value = ApiResult.Error("サーバーエラー: ${e.code()}")
        } catch (e: IOException) {
            _state.value = ApiResult.Error("ネットワーク接続を確認してください")
        }
    }
}
```

```kotlin
@Composable
fun WeatherScreen(viewModel: WeatherViewModel) {
    val state by viewModel.state.collectAsState()

    when (val result = state) {
        is ApiResult.Loading -> CircularProgressIndicator()
        is ApiResult.Error -> Text(result.message, color = MaterialTheme.colorScheme.error)
        is ApiResult.Success -> {
            Text("${result.data.name}: ${result.data.main.temp}°C")
        }
    }
}
```

---

## OkHttpインターセプター

ログ出力やAPIキーの自動付与に使います。

```kotlin
val client = OkHttpClient.Builder()
    .addInterceptor(HttpLoggingInterceptor().apply {
        level = HttpLoggingInterceptor.Level.BODY
    })
    .addInterceptor { chain ->
        val request = chain.request().newBuilder()
            .addHeader("Authorization", "Bearer $API_KEY")
            .build()
        chain.proceed(request)
    }
    .build()

val retrofit = Retrofit.Builder()
    .baseUrl(BASE_URL)
    .client(client)
    .addConverterFactory(GsonConverterFactory.create())
    .build()
```

---

## よく使うAPI例

| API | 用途 | 無料プラン |
|-----|------|----------|
| OpenWeatherMap | 天気情報 | 1,000回/日 |
| ExchangeRate-API | 為替レート | 1,500回/月 |
| News API | ニュース | 100回/日 |
| JSONPlaceholder | テスト用 | 無制限 |

---

## まとめ

- Retrofit = AndroidのAPI通信の標準ライブラリ
- 3ステップ：インターフェース定義 → インスタンス作成 → ViewModel呼び出し
- `suspend fun`でCoroutine連携
- `sealed class`でエラーハンドリング

---

8種類のAndroidアプリテンプレート（API連携の追加も簡単なMVVM設計）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [ViewModel + StateFlow入門](https://zenn.dev/myougatheaxo/articles/android-viewmodel-stateflow-2026)
- [Kotlin Coroutine入門](https://zenn.dev/myougatheaxo/articles/kotlin-coroutine-android-basics)
- [Compose Navigation入門](https://zenn.dev/myougatheaxo/articles/compose-navigation-guide-2026)
- [AIでAndroidアプリを47秒で作った話](https://zenn.dev/myougatheaxo/articles/ai-android-app-47seconds)
