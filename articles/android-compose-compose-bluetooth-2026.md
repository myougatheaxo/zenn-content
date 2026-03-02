---
title: "Compose Bluetooth完全ガイド — BLE接続/デバイススキャン/GATT通信/Compose UI"
emoji: "📶"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "bluetooth"]
published: true
---

## この記事で学べること

**Compose Bluetooth**（BLEスキャン、GATT接続、データ読み書き、Compose状態連携）を解説します。

---

## BLEスキャン

```kotlin
class BleScanner @Inject constructor(
    @ApplicationContext private val context: Context
) {
    private val bluetoothManager = context.getSystemService(BluetoothManager::class.java)
    private val scanner = bluetoothManager.adapter.bluetoothLeScanner
    private val _devices = MutableStateFlow<List<BleScanResult>>(emptyList())
    val devices = _devices.asStateFlow()

    @SuppressLint("MissingPermission")
    fun startScan() {
        val settings = ScanSettings.Builder()
            .setScanMode(ScanSettings.SCAN_MODE_LOW_LATENCY)
            .build()

        scanner.startScan(null, settings, object : ScanCallback() {
            override fun onScanResult(callbackType: Int, result: ScanResult) {
                val device = BleScanResult(
                    name = result.device.name ?: "Unknown",
                    address = result.device.address,
                    rssi = result.rssi
                )
                _devices.update { current ->
                    (current.filter { it.address != device.address } + device)
                        .sortedByDescending { it.rssi }
                }
            }
        })
    }

    @SuppressLint("MissingPermission")
    fun stopScan() { scanner.stopScan(null) }
}

data class BleScanResult(val name: String, val address: String, val rssi: Int)
```

---

## GATT接続

```kotlin
class BleConnection @Inject constructor(
    @ApplicationContext private val context: Context
) {
    private var gatt: BluetoothGatt? = null
    private val _state = MutableStateFlow(BleState.DISCONNECTED)
    val state = _state.asStateFlow()
    private val _data = MutableSharedFlow<ByteArray>()
    val data = _data.asSharedFlow()

    @SuppressLint("MissingPermission")
    fun connect(address: String) {
        val device = BluetoothAdapter.getDefaultAdapter().getRemoteDevice(address)
        _state.value = BleState.CONNECTING

        gatt = device.connectGatt(context, false, object : BluetoothGattCallback() {
            override fun onConnectionStateChange(gatt: BluetoothGatt, status: Int, newState: Int) {
                when (newState) {
                    BluetoothProfile.STATE_CONNECTED -> {
                        _state.value = BleState.CONNECTED
                        gatt.discoverServices()
                    }
                    BluetoothProfile.STATE_DISCONNECTED -> {
                        _state.value = BleState.DISCONNECTED
                    }
                }
            }

            override fun onCharacteristicChanged(
                gatt: BluetoothGatt, characteristic: BluetoothGattCharacteristic, value: ByteArray
            ) {
                _data.tryEmit(value)
            }
        })
    }

    @SuppressLint("MissingPermission")
    fun disconnect() { gatt?.close(); gatt = null }
}

enum class BleState { DISCONNECTED, CONNECTING, CONNECTED }
```

---

## Compose UI

```kotlin
@Composable
fun BleScreen(viewModel: BleViewModel = hiltViewModel()) {
    val devices by viewModel.devices.collectAsStateWithLifecycle()
    val state by viewModel.connectionState.collectAsStateWithLifecycle()

    Column(Modifier.fillMaxSize().padding(16.dp)) {
        Row(Modifier.fillMaxWidth(), horizontalArrangement = Arrangement.SpaceBetween) {
            Text("BLEデバイス", style = MaterialTheme.typography.headlineMedium)
            Button(onClick = { viewModel.toggleScan() }) {
                Text(if (viewModel.isScanning) "停止" else "スキャン")
            }
        }

        Spacer(Modifier.height(16.dp))

        LazyColumn {
            items(devices) { device ->
                Card(Modifier.fillMaxWidth().padding(vertical = 4.dp)) {
                    ListItem(
                        headlineContent = { Text(device.name) },
                        supportingContent = { Text(device.address) },
                        trailingContent = { Text("${device.rssi} dBm") },
                        modifier = Modifier.clickable { viewModel.connect(device.address) }
                    )
                }
            }
        }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `BluetoothLeScanner` | BLEスキャン |
| `BluetoothGatt` | GATT接続 |
| `ScanCallback` | デバイス検出 |
| `GattCallback` | データ通信 |

- `BluetoothLeScanner`でBLEデバイスをスキャン
- `BluetoothGatt`でGATT接続してデータ読み書き
- `StateFlow`でスキャン結果と接続状態をComposeに連携
- Android 12+は`BLUETOOTH_SCAN`/`BLUETOOTH_CONNECT`パーミッション

---

8種類のAndroidアプリテンプレート（Material3対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose NFC](https://zenn.dev/myougatheaxo/articles/android-compose-compose-nfc-2026)
- [Compose Permission](https://zenn.dev/myougatheaxo/articles/android-compose-compose-permission-2026)
- [Compose HealthConnect](https://zenn.dev/myougatheaxo/articles/android-compose-compose-health-connect-2026)
