---
title: "Room Prepopulate完全ガイド — 初期データ投入/アセットDB/Callback"
emoji: "📥"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "room"]
published: true
---

## この記事で学べること

**Room Prepopulate**（createFromAsset、createFromFile、RoomDatabase.Callback、初期データ）を解説します。

---

## createFromAsset

```kotlin
// assets/database/app.db に事前作成したDBを配置
Room.databaseBuilder(context, AppDatabase::class.java, "app.db")
    .createFromAsset("database/app.db")
    .build()

// マイグレーション付き
Room.databaseBuilder(context, AppDatabase::class.java, "app.db")
    .createFromAsset("database/app.db")
    .addMigrations(MIGRATION_1_2)
    .build()
```

---

## Callback

```kotlin
@Module
@InstallIn(SingletonComponent::class)
object DatabaseModule {
    @Provides
    @Singleton
    fun provideDatabase(@ApplicationContext context: Context): AppDatabase {
        return Room.databaseBuilder(context, AppDatabase::class.java, "app.db")
            .addCallback(object : RoomDatabase.Callback() {
                override fun onCreate(db: SupportSQLiteDatabase) {
                    super.onCreate(db)
                    // DB初回作成時のみ実行
                    CoroutineScope(Dispatchers.IO).launch {
                        val dao = provideDatabase(context).categoryDao()
                        dao.insertAll(getDefaultCategories())
                    }
                }
            })
            .build()
    }
}

fun getDefaultCategories() = listOf(
    Category(name = "仕事", color = "#4CAF50", icon = "work"),
    Category(name = "個人", color = "#2196F3", icon = "person"),
    Category(name = "買い物", color = "#FF9800", icon = "shopping"),
    Category(name = "健康", color = "#E91E63", icon = "health")
)
```

---

## createFromFile

```kotlin
// 外部ストレージからDBを復元
fun restoreDatabase(context: Context, backupFile: File): AppDatabase {
    return Room.databaseBuilder(context, AppDatabase::class.java, "app.db")
        .createFromFile(backupFile)
        .build()
}

// バックアップ+復元
class BackupManager @Inject constructor(
    @ApplicationContext private val context: Context,
    private val database: AppDatabase
) {
    fun backup(destination: File) {
        database.close()
        val dbFile = context.getDatabasePath("app.db")
        dbFile.copyTo(destination, overwrite = true)
    }

    fun restore(source: File) {
        database.close()
        val dbFile = context.getDatabasePath("app.db")
        source.copyTo(dbFile, overwrite = true)
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `createFromAsset` | アセットから初期DB |
| `createFromFile` | ファイルから復元 |
| `Callback.onCreate` | 初回作成時処理 |
| `Callback.onOpen` | DB開くたびに処理 |

- `createFromAsset`で事前作成DBを配布
- `Callback.onCreate`でプログラム的に初期データ投入
- `createFromFile`でバックアップから復元
- マイグレーションとの組み合わせも可能

---

8種類のAndroidアプリテンプレート（Room DB対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Room Migration](https://zenn.dev/myougatheaxo/articles/android-compose-room-migration-advanced-2026)
- [Room AutoMigration](https://zenn.dev/myougatheaxo/articles/android-compose-room-migration-auto-2026)
- [Room Backup](https://zenn.dev/myougatheaxo/articles/android-compose-room-backup-2026)
