---
title: "BLE (Bluetooth Low Energy) + Compose ガイド — スキャン/接続/データ通信"
emoji: "📡"
type: "tech"
topics: ["android", "kotlin", "bluetooth", "ble"]
published: true
---

## この記事で学べること

**BLE (Bluetooth Low Energy)** のCompose統合（デバイススキャン、GATT接続、Characteristic読み書き）を解説します。

---

## パーミッション設定

```xml
<!-- AndroidManifest.xml -->
<uses-permission android:name="android.permission.BLUETOOTH_SCAN" />
<uses-permission android:name="android.permission.BLUETOOTH_CONNECT" />
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
<uses-feature android:name="android.hardware.bluetooth_le" android:required="true" />
```

---

## BLEスキャン

```kotlin
class BleScanner(private val context: Context) {
    private val bluetoothAdapter = context.getSystemService<BluetoothManager>()?.adapter
    private val scanner = bluetoothAdapter?.bluetoothLeScanner

    private val _devices = MutableStateFlow<List<BluetoothDevice>>(emptyList())
    val devices: StateFlow<List<BluetoothDevice>> = _devices.asStateFlow()

    private val _isScanning = MutableStateFlow(false)
    val isScanning: StateFlow<Boolean> = _isScanning.asStateFlow()

    private val scanCallback = object : ScanCallback() {
        override fun onScanResult(callbackType: Int, result: ScanResult) {
            val device = result.device
            _devices.update { list ->
                if (list.none { it.address == device.address }) list + device else list
            }
        }
    }

    @SuppressLint("MissingPermission")
    fun startScan() {
        _devices.value = emptyList()
        _isScanning.value = true

        val filters = listOf(
            ScanFilter.Builder()
                .setServiceUuid(ParcelUuid(TARGET_SERVICE_UUID))
                .build()
        )
        val settings = ScanSettings.Builder()
            .setScanMode(ScanSettings.SCAN_MODE_LOW_LATENCY)
            .build()

        scanner?.startScan(filters, settings, scanCallback)

        // 10秒後に自動停止
        CoroutineScope(Dispatchers.Main).launch {
            delay(10_000)
            stopScan()
        }
    }

    @SuppressLint("MissingPermission")
    fun stopScan() {
        scanner?.stopScan(scanCallback)
        _isScanning.value = false
    }

    companion object {
        val TARGET_SERVICE_UUID: UUID = UUID.fromString("0000180d-0000-1000-8000-00805f9b34fb")
    }
}
```

---

## GATT接続

```kotlin
class BleConnection(private val context: Context) {
    private var gatt: BluetoothGatt? = null

    private val _connectionState = MutableStateFlow(ConnectionState.DISCONNECTED)
    val connectionState: StateFlow<ConnectionState> = _connectionState.asStateFlow()

    private val _receivedData = MutableSharedFlow<ByteArray>()
    val receivedData: SharedFlow<ByteArray> = _receivedData.asSharedFlow()

    @SuppressLint("MissingPermission")
    fun connect(device: BluetoothDevice) {
        _connectionState.value = ConnectionState.CONNECTING
        gatt = device.connectGatt(context, false, gattCallback)
    }

    private val gattCallback = object : BluetoothGattCallback() {
        override fun onConnectionStateChange(gatt: BluetoothGatt, status: Int, newState: Int) {
            when (newState) {
                BluetoothProfile.STATE_CONNECTED -> {
                    _connectionState.value = ConnectionState.CONNECTED
                    gatt.discoverServices()
                }
                BluetoothProfile.STATE_DISCONNECTED -> {
                    _connectionState.value = ConnectionState.DISCONNECTED
                }
            }
        }

        override fun onServicesDiscovered(gatt: BluetoothGatt, status: Int) {
            if (status == BluetoothGatt.GATT_SUCCESS) {
                enableNotifications(gatt)
            }
        }

        override fun onCharacteristicChanged(
            gatt: BluetoothGatt,
            characteristic: BluetoothGattCharacteristic,
            value: ByteArray
        ) {
            CoroutineScope(Dispatchers.IO).launch {
                _receivedData.emit(value)
            }
        }
    }

    @SuppressLint("MissingPermission")
    fun disconnect() {
        gatt?.disconnect()
        gatt?.close()
        gatt = null
    }
}

enum class ConnectionState { DISCONNECTED, CONNECTING, CONNECTED }
```

---

## Compose UI

```kotlin
@Composable
fun BleScreen() {
    val context = LocalContext.current
    val scanner = remember { BleScanner(context) }
    val devices by scanner.devices.collectAsStateWithLifecycle()
    val isScanning by scanner.isScanning.collectAsStateWithLifecycle()

    // パーミッション
    val permissionLauncher = rememberLauncherForActivityResult(
        ActivityResultContracts.RequestMultiplePermissions()
    ) { permissions ->
        if (permissions.values.all { it }) {
            scanner.startScan()
        }
    }

    Column(Modifier.padding(16.dp)) {
        Button(onClick = {
            permissionLauncher.launch(arrayOf(
                Manifest.permission.BLUETOOTH_SCAN,
                Manifest.permission.BLUETOOTH_CONNECT,
                Manifest.permission.ACCESS_FINE_LOCATION
            ))
        }) {
            if (isScanning) {
                CircularProgressIndicator(Modifier.size(16.dp), strokeWidth = 2.dp)
                Spacer(Modifier.width(8.dp))
                Text("スキャン中...")
            } else {
                Icon(Icons.Default.BluetoothSearching, null)
                Spacer(Modifier.width(8.dp))
                Text("スキャン開始")
            }
        }

        Spacer(Modifier.height(16.dp))

        LazyColumn {
            items(devices) { device ->
                DeviceItem(device = device, onClick = {
                    scanner.stopScan()
                    // 接続処理
                })
            }
        }
    }
}
```

---

## まとめ

- Android 12+は`BLUETOOTH_SCAN`/`BLUETOOTH_CONNECT`パーミッション
- `ScanFilter`でターゲットデバイスのみスキャン
- `BluetoothGattCallback`で接続/データ受信
- `MutableSharedFlow`でリアルタイムデータをCompose連携
- スキャンは10秒程度で自動停止（バッテリー節約）
- `disconnect()`+`close()`で確実にリソース解放

---

8種類のAndroidアプリテンプレート（ハードウェア連携対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [パーミッション処理](https://zenn.dev/myougatheaxo/articles/android-compose-permission-handling-2026)
- [Composeライフサイクル](https://zenn.dev/myougatheaxo/articles/android-compose-lifecycle-aware-2026)
- [Flow/Channel](https://zenn.dev/myougatheaxo/articles/kotlin-flow-channel-2026)
