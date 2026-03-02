---
title: "NFC読み取り完全ガイド — NDEF/タグ検出/Compose連携"
emoji: "📡"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "nfc"]
published: true
---

## この記事で学べること

**NFC読み取り**（NDEF読み書き、タグ検出、foreground dispatch、Compose UI連携）を解説します。

---

## マニフェスト設定

```xml
<manifest>
    <uses-permission android:name="android.permission.NFC" />
    <uses-feature android:name="android.hardware.nfc" android:required="true" />

    <activity android:name=".MainActivity">
        <intent-filter>
            <action android:name="android.nfc.action.NDEF_DISCOVERED" />
            <category android:name="android.intent.category.DEFAULT" />
            <data android:mimeType="text/plain" />
        </intent-filter>
    </activity>
</manifest>
```

---

## NfcManager

```kotlin
class NfcManager(private val activity: ComponentActivity) {
    private val nfcAdapter: NfcAdapter? = NfcAdapter.getDefaultAdapter(activity)

    val isNfcAvailable: Boolean get() = nfcAdapter != null
    val isNfcEnabled: Boolean get() = nfcAdapter?.isEnabled == true

    private val _tagData = MutableSharedFlow<NfcTagData>(replay = 1)
    val tagData: SharedFlow<NfcTagData> = _tagData

    fun enableForegroundDispatch() {
        val intent = Intent(activity, activity::class.java).addFlags(Intent.FLAG_ACTIVITY_SINGLE_TOP)
        val pendingIntent = PendingIntent.getActivity(activity, 0, intent, PendingIntent.FLAG_MUTABLE)
        val filters = arrayOf(IntentFilter(NfcAdapter.ACTION_NDEF_DISCOVERED).apply {
            addDataType("*/*")
        })
        nfcAdapter?.enableForegroundDispatch(activity, pendingIntent, filters, null)
    }

    fun disableForegroundDispatch() {
        nfcAdapter?.disableForegroundDispatch(activity)
    }

    fun handleIntent(intent: Intent) {
        if (intent.action == NfcAdapter.ACTION_NDEF_DISCOVERED ||
            intent.action == NfcAdapter.ACTION_TAG_DISCOVERED) {
            val tag = if (Build.VERSION.SDK_INT >= 33) {
                intent.getParcelableExtra(NfcAdapter.EXTRA_TAG, Tag::class.java)
            } else {
                @Suppress("DEPRECATION") intent.getParcelableExtra(NfcAdapter.EXTRA_TAG)
            }
            val messages = if (Build.VERSION.SDK_INT >= 33) {
                intent.getParcelableArrayExtra(NfcAdapter.EXTRA_NDEF_MESSAGES, NdefMessage::class.java)
            } else {
                @Suppress("DEPRECATION") intent.getParcelableArrayExtra(NfcAdapter.EXTRA_NDEF_MESSAGES)
            }

            val records = messages?.flatMap { (it as NdefMessage).records.toList() } ?: emptyList()
            val textContent = records.mapNotNull { record ->
                if (record.tnf == NdefRecord.TNF_WELL_KNOWN) String(record.payload, Charsets.UTF_8)
                else null
            }

            _tagData.tryEmit(NfcTagData(
                id = tag?.id?.toHexString() ?: "",
                techList = tag?.techList?.toList() ?: emptyList(),
                content = textContent
            ))
        }
    }
}

data class NfcTagData(
    val id: String,
    val techList: List<String>,
    val content: List<String>
)

fun ByteArray.toHexString() = joinToString("") { "%02X".format(it) }
```

---

## Compose画面

```kotlin
@Composable
fun NfcReaderScreen(nfcManager: NfcManager) {
    val tagData by nfcManager.tagData.collectAsStateWithLifecycle(initialValue = null)
    val lifecycleOwner = LocalLifecycleOwner.current

    DisposableEffect(lifecycleOwner) {
        val observer = LifecycleEventObserver { _, event ->
            when (event) {
                Lifecycle.Event.ON_RESUME -> nfcManager.enableForegroundDispatch()
                Lifecycle.Event.ON_PAUSE -> nfcManager.disableForegroundDispatch()
                else -> {}
            }
        }
        lifecycleOwner.lifecycle.addObserver(observer)
        onDispose { lifecycleOwner.lifecycle.removeObserver(observer) }
    }

    Column(
        Modifier.fillMaxSize().padding(16.dp),
        horizontalAlignment = Alignment.CenterHorizontally
    ) {
        if (!nfcManager.isNfcAvailable) {
            Text("このデバイスはNFCに対応していません", color = MaterialTheme.colorScheme.error)
        } else if (!nfcManager.isNfcEnabled) {
            Text("NFCが無効です。設定で有効にしてください")
        } else {
            Icon(Icons.Default.Nfc, null, Modifier.size(64.dp), tint = MaterialTheme.colorScheme.primary)
            Text("NFCタグをかざしてください", style = MaterialTheme.typography.headlineSmall)
        }

        tagData?.let { data ->
            Spacer(Modifier.height(24.dp))
            Card(Modifier.fillMaxWidth()) {
                Column(Modifier.padding(16.dp)) {
                    Text("タグID: ${data.id}", style = MaterialTheme.typography.titleMedium)
                    Text("技術: ${data.techList.joinToString()}", style = MaterialTheme.typography.bodySmall)
                    data.content.forEach { Text("内容: $it") }
                }
            }
        }
    }
}
```

---

## まとめ

| 機能 | 実装 |
|------|------|
| タグ検出 | `enableForegroundDispatch` |
| NDEF読み取り | `NdefMessage.records` |
| NFC状態 | `NfcAdapter.isEnabled` |
| Compose連携 | `SharedFlow` + `collectAsState` |

- `enableForegroundDispatch`でアプリ前面時のタグ検出
- `NdefRecord`からテキスト/URI/MIMEデータ抽出
- Lifecycle連動でresume/pause時に自動切替
- NFCの利用可否をUIで適切に表示

---

8種類のAndroidアプリテンプレート（ハードウェア連携対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Bluetooth BLE](https://zenn.dev/myougatheaxo/articles/android-compose-bluetooth-ble-2026)
- [バーコードスキャナー](https://zenn.dev/myougatheaxo/articles/android-compose-barcode-scanner-2026)
- [カメラ](https://zenn.dev/myougatheaxo/articles/android-compose-camera-capture-2026)
