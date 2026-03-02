---
title: "ContentProvider完全ガイド — データ共有/URI設計/FileProvider"
emoji: "🗃️"
type: "tech"
topics: ["android", "kotlin", "contentprovider", "architecture"]
published: true
---

## この記事で学べること

**ContentProvider**（データ共有、URI設計、FileProvider、ContactsProvider連携）を解説します。

---

## カスタムContentProvider

```kotlin
class NoteContentProvider : ContentProvider() {
    private lateinit var db: NoteDatabase

    companion object {
        const val AUTHORITY = "com.example.app.provider"
        val CONTENT_URI: Uri = Uri.parse("content://$AUTHORITY/notes")
    }

    override fun onCreate(): Boolean {
        db = Room.databaseBuilder(context!!, NoteDatabase::class.java, "notes.db").build()
        return true
    }

    override fun query(
        uri: Uri, projection: Array<String>?, selection: String?,
        selectionArgs: Array<String>?, sortOrder: String?
    ): Cursor? {
        val cursor = db.noteDao().getAllAsCursor()
        cursor.setNotificationUri(context!!.contentResolver, uri)
        return cursor
    }

    override fun insert(uri: Uri, values: ContentValues?): Uri? {
        val id = db.noteDao().insert(Note.fromContentValues(values!!))
        context!!.contentResolver.notifyChange(uri, null)
        return ContentUris.withAppendedId(CONTENT_URI, id)
    }

    override fun update(uri: Uri, values: ContentValues?, selection: String?, selectionArgs: Array<String>?): Int {
        val count = db.noteDao().update(Note.fromContentValues(values!!))
        context!!.contentResolver.notifyChange(uri, null)
        return count
    }

    override fun delete(uri: Uri, selection: String?, selectionArgs: Array<String>?): Int {
        val id = ContentUris.parseId(uri)
        val count = db.noteDao().deleteById(id)
        context!!.contentResolver.notifyChange(uri, null)
        return count
    }

    override fun getType(uri: Uri): String = "vnd.android.cursor.dir/vnd.example.notes"
}
```

---

## FileProvider

```xml
<!-- AndroidManifest.xml -->
<provider
    android:name="androidx.core.content.FileProvider"
    android:authorities="${applicationId}.fileprovider"
    android:exported="false"
    android:grantUriPermissions="true">
    <meta-data
        android:name="android.support.FILE_PROVIDER_PATHS"
        android:resource="@xml/file_paths" />
</provider>
```

```xml
<!-- res/xml/file_paths.xml -->
<paths>
    <cache-path name="cache" path="." />
    <files-path name="files" path="." />
    <external-files-path name="external" path="." />
</paths>
```

```kotlin
// ファイル共有
fun shareFile(context: Context, file: File) {
    val uri = FileProvider.getUriForFile(
        context, "${context.packageName}.fileprovider", file
    )

    val intent = Intent(Intent.ACTION_SEND).apply {
        type = "application/pdf"
        putExtra(Intent.EXTRA_STREAM, uri)
        addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION)
    }

    context.startActivity(Intent.createChooser(intent, "共有"))
}
```

---

## 連絡先取得

```kotlin
@Composable
fun ContactPicker() {
    val context = LocalContext.current
    var contacts by remember { mutableStateOf<List<Contact>>(emptyList()) }

    val pickContact = rememberLauncherForActivityResult(
        ActivityResultContracts.PickContact()
    ) { uri ->
        uri?.let {
            val cursor = context.contentResolver.query(it, null, null, null, null)
            cursor?.use { c ->
                if (c.moveToFirst()) {
                    val name = c.getString(c.getColumnIndexOrThrow(ContactsContract.Contacts.DISPLAY_NAME))
                    contacts = contacts + Contact(name)
                }
            }
        }
    }

    Button(onClick = { pickContact.launch(null) }) { Text("連絡先を選択") }
}
```

---

## まとめ

| 機能 | 用途 |
|------|------|
| `ContentProvider` | アプリ間データ共有 |
| `FileProvider` | ファイルURI安全共有 |
| `ContentResolver` | Provider問い合わせ |
| `PickContact` | 連絡先選択 |

- `ContentProvider`でアプリ間のデータ共有
- `FileProvider`でファイルを安全に共有（file:// URI禁止対応）
- `ContentResolver`で他アプリのデータにアクセス
- `notifyChange`でデータ変更を通知

---

8種類のAndroidアプリテンプレート（データ共有対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Room CRUD](https://zenn.dev/myougatheaxo/articles/android-compose-room-flow-crud-2026)
- [Share Intent](https://zenn.dev/myougatheaxo/articles/android-compose-share-intent-2026)
- [ファイル管理](https://zenn.dev/myougatheaxo/articles/android-compose-file-manager-2026)
