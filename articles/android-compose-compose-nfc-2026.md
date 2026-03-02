---
title: "Compose NFC完全ガイド — NFCタグ読み書き/NDEF/HCE/カードエミュレーション"
emoji: "📡"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "nfc"]
published: true
---

## この記事で学べること

**Compose NFC**（NFCタグ読み取り、NDEFメッセージ、タグ書き込み、カードエミュレーション）を解説します。

---

## NFC読み取り

```kotlin
class MainActivity : ComponentActivity() {
    private var nfcAdapter: NfcAdapter? = null

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        nfcAdapter = NfcAdapter.getDefaultAdapter(this)
        setContent {
            MaterialTheme { NfcScreen() }
        }
    }

    override fun onResume() {
        super.onResume()
        val intent = PendingIntent.getActivity(this, 0,
            Intent(this, javaClass).addFlags(Intent.FLAG_ACTIVITY_SINGLE_TOP),
            PendingIntent.FLAG_MUTABLE)
        nfcAdapter?.enableForegroundDispatch(this, intent, null, null)
    }

    override fun onPause() {
        super.onPause()
        nfcAdapter?.disableForegroundDispatch(this)
    }

    override fun onNewIntent(intent: Intent) {
        super.onNewIntent(intent)
        if (NfcAdapter.ACTION_NDEF_DISCOVERED == intent.action ||
            NfcAdapter.ACTION_TAG_DISCOVERED == intent.action) {
            val tag = intent.getParcelableExtra<Tag>(NfcAdapter.EXTRA_TAG)
            // タグデータを処理
        }
    }
}
```

---

## Compose UI

```kotlin
@Composable
fun NfcScreen() {
    val context = LocalContext.current
    val nfcAvailable = remember { NfcAdapter.getDefaultAdapter(context) != null }
    var tagData by remember { mutableStateOf<String?>(null) }

    Column(Modifier.fillMaxSize().padding(16.dp),
        horizontalAlignment = Alignment.CenterHorizontally,
        verticalArrangement = Arrangement.Center) {

        if (!nfcAvailable) {
            Icon(Icons.Default.Warning, null, Modifier.size(64.dp), tint = Color.Red)
            Text("NFCが利用できません", style = MaterialTheme.typography.titleLarge)
        } else {
            Icon(Icons.Default.Nfc, null, Modifier.size(96.dp),
                tint = MaterialTheme.colorScheme.primary)
            Spacer(Modifier.height(16.dp))
            Text("NFCタグをかざしてください", style = MaterialTheme.typography.titleMedium)

            tagData?.let {
                Spacer(Modifier.height(24.dp))
                ElevatedCard(Modifier.fillMaxWidth()) {
                    Column(Modifier.padding(16.dp)) {
                        Text("読み取りデータ", style = MaterialTheme.typography.titleSmall)
                        Text(it, style = MaterialTheme.typography.bodyLarge)
                    }
                }
            }
        }
    }
}

fun parseNdefMessage(intent: Intent): String? {
    val messages = intent.getParcelableArrayExtra(NfcAdapter.EXTRA_NDEF_MESSAGES)
    return messages?.firstOrNull()?.let { raw ->
        val msg = raw as NdefMessage
        msg.records.firstOrNull()?.let { record ->
            String(record.payload, Charsets.UTF_8).drop(3) // 言語コードをスキップ
        }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `NfcAdapter` | NFC検出 |
| `enableForegroundDispatch` | フォアグラウンド受信 |
| `NdefMessage` | NDEFデータ |
| `Tag` | NFCタグ情報 |

- `enableForegroundDispatch`でアプリがフォアグラウンド時にNFC受信
- `NdefMessage`でテキスト/URI/MIMEデータを解析
- `onNewIntent`でタグ検出を処理
- ManifestにNFCパーミッションと`uses-feature`を追加

---

8種類のAndroidアプリテンプレート（Material3対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose BroadcastReceiver](https://zenn.dev/myougatheaxo/articles/android-compose-compose-broadcast-receiver-2026)
- [Compose Connectivity](https://zenn.dev/myougatheaxo/articles/android-compose-compose-connectivity-2026)
- [Compose Permissions](https://zenn.dev/myougatheaxo/articles/android-compose-compose-permissions-2026)
