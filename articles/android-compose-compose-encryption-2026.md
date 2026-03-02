---
title: "Compose Encryption完全ガイド — EncryptedSharedPreferences/Keystore/AES暗号化/セキュアストレージ"
emoji: "🔐"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "security"]
published: true
---

## この記事で学べること

**Compose Encryption**（EncryptedSharedPreferences、Android Keystore、AES暗号化、セキュアなデータ保存）を解説します。

---

## EncryptedSharedPreferences

```kotlin
class SecureStorage @Inject constructor(
    @ApplicationContext private val context: Context
) {
    private val prefs = EncryptedSharedPreferences.create(
        context,
        "secure_prefs",
        MasterKeys.getOrCreate(MasterKeys.AES256_GCM_SPEC),
        EncryptedSharedPreferences.PrefKeyEncryptionScheme.AES256_SIV,
        EncryptedSharedPreferences.PrefValueEncryptionScheme.AES256_GCM
    )

    fun saveToken(token: String) { prefs.edit().putString("auth_token", token).apply() }
    fun getToken(): String? = prefs.getString("auth_token", null)
    fun clearToken() { prefs.edit().remove("auth_token").apply() }
}

// Compose連携
@Composable
fun SecureSettingsScreen(storage: SecureStorage) {
    var token by remember { mutableStateOf(storage.getToken() ?: "") }

    Column(Modifier.padding(16.dp)) {
        OutlinedTextField(
            value = token,
            onValueChange = { token = it },
            label = { Text("APIトークン") },
            visualTransformation = PasswordVisualTransformation()
        )
        Spacer(Modifier.height(16.dp))
        Button(onClick = { storage.saveToken(token) }) { Text("保存（暗号化）") }
    }
}
```

---

## Keystoreで暗号化

```kotlin
class KeystoreEncryptor {
    private val keyAlias = "app_encryption_key"

    private fun getKey(): SecretKey {
        val keyStore = KeyStore.getInstance("AndroidKeyStore").apply { load(null) }
        return if (keyStore.containsAlias(keyAlias)) {
            (keyStore.getEntry(keyAlias, null) as KeyStore.SecretKeyEntry).secretKey
        } else {
            val keyGen = KeyGenerator.getInstance(KeyProperties.KEY_ALGORITHM_AES, "AndroidKeyStore")
            keyGen.init(
                KeyGenParameterSpec.Builder(keyAlias,
                    KeyProperties.PURPOSE_ENCRYPT or KeyProperties.PURPOSE_DECRYPT)
                    .setBlockModes(KeyProperties.BLOCK_MODE_GCM)
                    .setEncryptionPaddings(KeyProperties.ENCRYPTION_PADDING_NONE)
                    .build()
            )
            keyGen.generateKey()
        }
    }

    fun encrypt(data: ByteArray): Pair<ByteArray, ByteArray> {
        val cipher = Cipher.getInstance("AES/GCM/NoPadding")
        cipher.init(Cipher.ENCRYPT_MODE, getKey())
        return cipher.iv to cipher.doFinal(data)
    }

    fun decrypt(iv: ByteArray, encrypted: ByteArray): ByteArray {
        val cipher = Cipher.getInstance("AES/GCM/NoPadding")
        cipher.init(Cipher.DECRYPT_MODE, getKey(), GCMParameterSpec(128, iv))
        return cipher.doFinal(encrypted)
    }
}
```

---

## ファイル暗号化

```kotlin
class EncryptedFileManager @Inject constructor(
    @ApplicationContext private val context: Context
) {
    private val masterKey = MasterKeys.getOrCreate(MasterKeys.AES256_GCM_SPEC)

    fun writeEncryptedFile(filename: String, content: String) {
        val file = File(context.filesDir, filename)
        val encryptedFile = EncryptedFile.Builder(
            file, context, masterKey,
            EncryptedFile.FileEncryptionScheme.AES256_GCM_HKDF_4KB
        ).build()

        encryptedFile.openFileOutput().use { it.write(content.toByteArray()) }
    }

    fun readEncryptedFile(filename: String): String {
        val file = File(context.filesDir, filename)
        val encryptedFile = EncryptedFile.Builder(
            file, context, masterKey,
            EncryptedFile.FileEncryptionScheme.AES256_GCM_HKDF_4KB
        ).build()

        return encryptedFile.openFileInput().bufferedReader().readText()
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `EncryptedSharedPreferences` | 暗号化KV保存 |
| `Android Keystore` | 鍵管理 |
| `EncryptedFile` | ファイル暗号化 |
| `MasterKeys` | マスターキー生成 |

- `EncryptedSharedPreferences`でキー/値を自動暗号化
- Android Keystoreでハードウェアベースの鍵管理
- `EncryptedFile`でファイル全体を暗号化
- AES256-GCMによる強力な暗号化

---

8種類のAndroidアプリテンプレート（セキュリティ対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose Biometric](https://zenn.dev/myougatheaxo/articles/android-compose-compose-biometric-2026)
- [Compose DataStore](https://zenn.dev/myougatheaxo/articles/android-compose-compose-datastore-2026)
- [Compose PlayIntegrity](https://zenn.dev/myougatheaxo/articles/android-compose-compose-play-integrity-2026)
