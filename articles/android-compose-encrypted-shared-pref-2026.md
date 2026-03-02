---
title: "暗号化ストレージ完全ガイド — EncryptedSharedPreferences/KeyStore/セキュア保存"
emoji: "🔒"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "security"]
published: true
---

## この記事で学べること

**暗号化ストレージ**（EncryptedSharedPreferences、AndroidKeyStore、トークン保存、セキュアな設定管理）を解説します。

---

## EncryptedSharedPreferences

```kotlin
class SecureStorage @Inject constructor(
    @ApplicationContext private val context: Context
) {
    private val masterKey = MasterKey.Builder(context)
        .setKeyScheme(MasterKey.KeyScheme.AES256_GCM)
        .build()

    private val encryptedPrefs = EncryptedSharedPreferences.create(
        context,
        "secure_prefs",
        masterKey,
        EncryptedSharedPreferences.PrefKeyEncryptionScheme.AES256_SIV,
        EncryptedSharedPreferences.PrefValueEncryptionScheme.AES256_GCM
    )

    fun saveToken(token: String) {
        encryptedPrefs.edit().putString("auth_token", token).apply()
    }

    fun getToken(): String? {
        return encryptedPrefs.getString("auth_token", null)
    }

    fun saveApiKey(key: String) {
        encryptedPrefs.edit().putString("api_key", key).apply()
    }

    fun clearAll() {
        encryptedPrefs.edit().clear().apply()
    }
}
```

---

## KeyStore暗号化

```kotlin
class KeyStoreHelper {
    private val keyStore = KeyStore.getInstance("AndroidKeyStore").apply { load(null) }

    fun encrypt(data: String, alias: String): ByteArray {
        val keyGenerator = KeyGenerator.getInstance(KeyProperties.KEY_ALGORITHM_AES, "AndroidKeyStore")
        if (!keyStore.containsAlias(alias)) {
            keyGenerator.init(
                KeyGenParameterSpec.Builder(alias, KeyProperties.PURPOSE_ENCRYPT or KeyProperties.PURPOSE_DECRYPT)
                    .setBlockModes(KeyProperties.BLOCK_MODE_GCM)
                    .setEncryptionPaddings(KeyProperties.ENCRYPTION_PADDING_NONE)
                    .build()
            )
            keyGenerator.generateKey()
        }

        val key = keyStore.getKey(alias, null) as SecretKey
        val cipher = Cipher.getInstance("AES/GCM/NoPadding")
        cipher.init(Cipher.ENCRYPT_MODE, key)

        val iv = cipher.iv
        val encrypted = cipher.doFinal(data.toByteArray())
        return iv + encrypted  // IV + 暗号文
    }

    fun decrypt(data: ByteArray, alias: String): String {
        val key = keyStore.getKey(alias, null) as SecretKey
        val cipher = Cipher.getInstance("AES/GCM/NoPadding")
        val iv = data.copyOfRange(0, 12)
        val encrypted = data.copyOfRange(12, data.size)

        cipher.init(Cipher.DECRYPT_MODE, key, GCMParameterSpec(128, iv))
        return String(cipher.doFinal(encrypted))
    }
}
```

---

## Compose設定画面

```kotlin
@Composable
fun SecureSettingsScreen(viewModel: SecureViewModel = hiltViewModel()) {
    val hasToken by viewModel.hasToken.collectAsStateWithLifecycle(false)

    Column(Modifier.fillMaxSize().padding(16.dp)) {
        Text("セキュリティ設定", style = MaterialTheme.typography.headlineSmall)
        Spacer(Modifier.height(16.dp))

        ListItem(
            headlineContent = { Text("認証トークン") },
            supportingContent = { Text(if (hasToken) "保存済み" else "未設定") },
            leadingContent = { Icon(Icons.Default.VpnKey, null) },
            trailingContent = {
                if (hasToken) {
                    IconButton(onClick = { viewModel.clearToken() }) {
                        Icon(Icons.Default.Delete, "削除")
                    }
                }
            }
        )

        HorizontalDivider()

        ListItem(
            headlineContent = { Text("全データ消去") },
            supportingContent = { Text("暗号化ストレージを初期化") },
            leadingContent = { Icon(Icons.Default.DeleteForever, null, tint = MaterialTheme.colorScheme.error) },
            modifier = Modifier.clickable { viewModel.clearAll() }
        )
    }
}
```

---

## まとめ

| 手法 | 用途 |
|------|------|
| EncryptedSharedPreferences | トークン/APIキー |
| AndroidKeyStore | 暗号化キー管理 |
| AES256_GCM | 値の暗号化 |
| AES256_SIV | キーの暗号化 |

- `EncryptedSharedPreferences`で透過的な暗号化
- `MasterKey`でAES256暗号化キーを自動管理
- `AndroidKeyStore`でハードウェアバックドキー保管
- 認証トークン/APIキーは必ず暗号化保存

---

8種類のAndroidアプリテンプレート（セキュリティ対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [生体認証](https://zenn.dev/myougatheaxo/articles/android-compose-biometric-prompt-2026)
- [Certificate Pinning](https://zenn.dev/myougatheaxo/articles/android-compose-certificate-pinning-2026)
- [DataStore](https://zenn.dev/myougatheaxo/articles/android-compose-datastore-preferences-2026)
