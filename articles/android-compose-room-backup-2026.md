---
title: "Room Backup完全ガイド — DBバックアップ/復元/エクスポート/インポート"
emoji: "💾"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "room"]
published: true
---

## この記事で学べること

**Room Backup**（データベースのバックアップ、復元、エクスポート、インポート）を解説します。

---

## DBバックアップ

```kotlin
class DatabaseBackup @Inject constructor(
    @ApplicationContext private val context: Context,
    private val database: AppDatabase
) {
    fun backup(): File {
        database.runInTransaction {} // WALチェックポイント
        val dbFile = context.getDatabasePath("app.db")
        val backupDir = File(context.filesDir, "backups").apply { mkdirs() }
        val backupFile = File(backupDir, "backup_${System.currentTimeMillis()}.db")
        dbFile.copyTo(backupFile, overwrite = true)
        // WAL/SHMファイルも
        File("${dbFile.path}-wal").let { if (it.exists()) it.copyTo(File("${backupFile.path}-wal"), true) }
        File("${dbFile.path}-shm").let { if (it.exists()) it.copyTo(File("${backupFile.path}-shm"), true) }
        return backupFile
    }

    suspend fun restore(backupFile: File) {
        database.close()
        val dbFile = context.getDatabasePath("app.db")
        backupFile.copyTo(dbFile, overwrite = true)
    }

    fun getBackups(): List<File> {
        return File(context.filesDir, "backups")
            .listFiles { f -> f.extension == "db" }
            ?.sortedByDescending { it.lastModified() }
            ?: emptyList()
    }
}
```

---

## 共有バックアップ

```kotlin
@Composable
fun BackupScreen(backup: DatabaseBackup) {
    val context = LocalContext.current
    val exportLauncher = rememberLauncherForActivityResult(
        ActivityResultContracts.CreateDocument("application/octet-stream")
    ) { uri ->
        uri?.let {
            val backupFile = backup.backup()
            context.contentResolver.openOutputStream(uri)?.use { out ->
                backupFile.inputStream().use { it.copyTo(out) }
            }
        }
    }

    val importLauncher = rememberLauncherForActivityResult(
        ActivityResultContracts.OpenDocument()
    ) { uri ->
        uri?.let {
            val tempFile = File(context.cacheDir, "import.db")
            context.contentResolver.openInputStream(uri)?.use { input ->
                tempFile.outputStream().use { input.copyTo(it) }
            }
            CoroutineScope(Dispatchers.IO).launch { backup.restore(tempFile) }
        }
    }

    Column(Modifier.padding(16.dp), verticalArrangement = Arrangement.spacedBy(12.dp)) {
        Button(onClick = { exportLauncher.launch("backup.db") }, Modifier.fillMaxWidth()) {
            Icon(Icons.Default.Upload, null); Spacer(Modifier.width(8.dp)); Text("エクスポート")
        }
        OutlinedButton(onClick = { importLauncher.launch(arrayOf("*/*")) }, Modifier.fillMaxWidth()) {
            Icon(Icons.Default.Download, null); Spacer(Modifier.width(8.dp)); Text("インポート")
        }
    }
}
```

---

## まとめ

| 操作 | 方法 |
|------|------|
| バックアップ | ファイルコピー |
| 復元 | ファイル上書き |
| エクスポート | SAF CreateDocument |
| インポート | SAF OpenDocument |

- `copyTo`でDBファイルをバックアップ
- WAL/SHMファイルも忘れずにコピー
- SAFで外部ストレージへエクスポート/インポート
- 復元前に`database.close()`必須

---

8種類のAndroidアプリテンプレート（バックアップ対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Room Prepopulate](https://zenn.dev/myougatheaxo/articles/android-compose-room-prepopulate-2026)
- [Room Migration](https://zenn.dev/myougatheaxo/articles/android-compose-room-migration-advanced-2026)
- [DataStore](https://zenn.dev/myougatheaxo/articles/android-compose-datastore-2026)
