---
title: "Room Encryption完全ガイド — SQLCipher/DB暗号化/セキュアストレージ"
emoji: "🔐"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "room"]
published: true
---

## この記事で学べること

**Room Encryption**（SQLCipher、DB暗号化、パスフレーズ管理、セキュアストレージ）を解説します。

---

## SQLCipher設定

```kotlin
// build.gradle
// implementation "net.zetetic:android-database-sqlcipher:4.5.4"
// implementation "androidx.sqlite:sqlite-ktx:2.4.0"

@Module
@InstallIn(SingletonComponent::class)
object DatabaseModule {
    @Provides
    @Singleton
    fun provideDatabase(
        @ApplicationContext context: Context,
        passphraseManager: PassphraseManager
    ): AppDatabase {
        val passphrase = passphraseManager.getPassphrase()
        val factory = SupportOpenHelperFactory(passphrase)

        return Room.databaseBuilder(context, AppDatabase::class.java, "encrypted.db")
            .openHelperFactory(factory)
            .build()
    }
}
```

---

## パスフレーズ管理

```kotlin
class PassphraseManager @Inject constructor(
    @ApplicationContext private val context: Context
) {
    private val keyStore = KeyStore.getInstance("AndroidKeyStore").apply { load(null) }
    private val alias = "db_passphrase_key"

    fun getPassphrase(): ByteArray {
        val prefs = context.getSharedPreferences("db_prefs", Context.MODE_PRIVATE)
        val stored = prefs.getString("encrypted_passphrase", null)

        return if (stored != null) {
            decrypt(Base64.decode(stored, Base64.DEFAULT))
        } else {
            val passphrase = generatePassphrase()
            val encrypted = encrypt(passphrase)
            prefs.edit().putString("encrypted_passphrase", Base64.encodeToString(encrypted, Base64.DEFAULT)).apply()
            passphrase
        }
    }

    private fun generatePassphrase(): ByteArray {
        val bytes = ByteArray(32)
        SecureRandom().nextBytes(bytes)
        return bytes
    }

    private fun encrypt(data: ByteArray): ByteArray {
        val cipher = Cipher.getInstance("AES/GCM/NoPadding")
        cipher.init(Cipher.ENCRYPT_MODE, getOrCreateKey())
        val iv = cipher.iv
        val encrypted = cipher.doFinal(data)
        return iv + encrypted
    }

    private fun decrypt(data: ByteArray): ByteArray {
        val cipher = Cipher.getInstance("AES/GCM/NoPadding")
        val iv = data.take(12).toByteArray()
        val encrypted = data.drop(12).toByteArray()
        cipher.init(Cipher.DECRYPT_MODE, getOrCreateKey(), GCMParameterSpec(128, iv))
        return cipher.doFinal(encrypted)
    }

    private fun getOrCreateKey(): SecretKey {
        if (!keyStore.containsAlias(alias)) {
            val spec = KeyGenParameterSpec.Builder(alias, KeyProperties.PURPOSE_ENCRYPT or KeyProperties.PURPOSE_DECRYPT)
                .setBlockModes(KeyProperties.BLOCK_MODE_GCM)
                .setEncryptionPaddings(KeyProperties.ENCRYPTION_PADDING_NONE)
                .build()
            KeyGenerator.getInstance(KeyProperties.KEY_ALGORITHM_AES, "AndroidKeyStore").apply {
                init(spec); generateKey()
            }
        }
        return keyStore.getKey(alias, null) as SecretKey
    }
}
```

---

## 暗号化の確認

```kotlin
@Composable
fun EncryptionStatus(database: AppDatabase) {
    var isEncrypted by remember { mutableStateOf(false) }

    LaunchedEffect(Unit) {
        withContext(Dispatchers.IO) {
            try {
                database.openHelper.readableDatabase
                isEncrypted = true
            } catch (e: Exception) {
                isEncrypted = false
            }
        }
    }

    ListItem(
        headlineContent = { Text("データベース暗号化") },
        trailingContent = {
            Icon(
                if (isEncrypted) Icons.Default.Lock else Icons.Default.LockOpen,
                null, tint = if (isEncrypted) Color.Green else Color.Red
            )
        }
    )
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `SQLCipher` | DB暗号化 |
| `SupportOpenHelperFactory` | Room統合 |
| `AndroidKeyStore` | 鍵管理 |
| `SecureRandom` | パスフレーズ生成 |

- `SQLCipher`でRoom DBを透過的に暗号化
- パスフレーズは`AndroidKeyStore`で安全に保管
- 暗号化DBはファイルコピーされても読めない
- パフォーマンスへの影響は軽微（5-10%程度）

---

8種類のAndroidアプリテンプレート（セキュリティ対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Room Backup](https://zenn.dev/myougatheaxo/articles/android-compose-room-backup-2026)
- [Compose Biometric](https://zenn.dev/myougatheaxo/articles/android-compose-compose-biometric-2026)
- [DataStore](https://zenn.dev/myougatheaxo/articles/android-compose-datastore-2026)
