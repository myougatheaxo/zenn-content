---
title: "Room Inspector完全ガイド — Database Inspector/デバッグ/クエリ実行"
emoji: "🔍"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "room"]
published: true
---

## この記事で学べること

**Room Inspector**（Android Studio Database Inspector、デバッグ用DAO、クエリ実行、データ確認）を解説します。

---

## Database Inspector

```kotlin
// Android Studioで: View > Tool Windows > App Inspection > Database Inspector
// API 26+のデバイスで実行中のアプリのDBをリアルタイム確認

// デバッグビルドでのみInspector対応
@Database(entities = [Task::class, User::class], version = 1)
abstract class AppDatabase : RoomDatabase() {
    abstract fun taskDao(): TaskDao
    abstract fun userDao(): UserDao
}

// build.gradle (デバッグ時のみ)
// debugImplementation "androidx.room:room-testing:2.6.1"
```

---

## デバッグ用DAO

```kotlin
@Dao
interface DebugDao {
    @RawQuery
    suspend fun rawQuery(query: SupportSQLiteQuery): List<Any>

    @Query("SELECT COUNT(*) FROM Task")
    suspend fun getTaskCount(): Int

    @Query("SELECT COUNT(*) FROM User")
    suspend fun getUserCount(): Int

    @Query("SELECT name FROM sqlite_master WHERE type='table'")
    suspend fun getAllTableNames(): List<String>
}

// デバッグメニュー
@Composable
fun DebugScreen(database: AppDatabase) {
    val scope = rememberCoroutineScope()
    var tableInfo by remember { mutableStateOf<List<String>>(emptyList()) }
    var taskCount by remember { mutableIntStateOf(0) }

    LaunchedEffect(Unit) {
        withContext(Dispatchers.IO) {
            tableInfo = database.debugDao().getAllTableNames()
            taskCount = database.debugDao().getTaskCount()
        }
    }

    Column(Modifier.padding(16.dp)) {
        Text("DB デバッグ", style = MaterialTheme.typography.titleLarge)
        Text("タスク数: $taskCount")
        Text("テーブル一覧:")
        tableInfo.forEach { Text("  - $it") }

        Button(onClick = {
            scope.launch(Dispatchers.IO) {
                database.clearAllTables()
                taskCount = 0
            }
        }) { Text("全データ削除") }
    }
}
```

---

## Stetho連携

```kotlin
// build.gradle
// debugImplementation "com.facebook.stetho:stetho:1.6.0"

class DebugApplication : Application() {
    override fun onCreate() {
        super.onCreate()
        if (BuildConfig.DEBUG) {
            Stetho.initializeWithDefaults(this)
        }
    }
}
// Chrome: chrome://inspect → Database確認
```

---

## まとめ

| ツール | 用途 |
|--------|------|
| Database Inspector | AS内リアルタイム確認 |
| `@RawQuery` | 任意SQL実行 |
| Stetho | Chrome DevTools連携 |
| `clearAllTables()` | 全データ削除 |

- Database InspectorはAndroid Studio標準ツール
- API 26+で実行中のアプリのDBをリアルタイム確認
- デバッグ用DAOは`debugImplementation`で分離
- `BuildConfig.DEBUG`でリリースビルドから除外

---

8種類のAndroidアプリテンプレート（Room DB対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Room Testing](https://zenn.dev/myougatheaxo/articles/android-compose-room-testing-2026)
- [Room Migration](https://zenn.dev/myougatheaxo/articles/android-compose-room-migration-advanced-2026)
- [Room RawQuery](https://zenn.dev/myougatheaxo/articles/android-compose-room-rawquery-2026)
