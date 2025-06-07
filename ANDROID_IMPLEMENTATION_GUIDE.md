# Android Implementation Guide - Bluetooth Audio Multicast

## Development Environment Setup

### Requirements
- **Android Studio Arctic Fox or newer**
- **Minimum SDK: API 31 (Android 12)** for LE Audio support
- **Target SDK: API 34 (Android 14)**
- **Physical devices** for Bluetooth testing
- **Google Play Console account** for distribution

### Project Configuration
```gradle
// app/build.gradle
android {
    compileSdk 34
    
    defaultConfig {
        minSdk 31
        targetSdk 34
        versionCode 1
        versionName "1.0"
    }
    
    compileOptions {
        sourceCompatibility JavaVersion.VERSION_1_8
        targetCompatibility JavaVersion.VERSION_1_8
    }
}

dependencies {
    implementation 'androidx.core:core:1.12.0'
    implementation 'androidx.appcompat:appcompat:1.6.1'
    implementation 'com.google.android.material:material:1.10.0'
    implementation 'androidx.constraintlayout:constraintlayout:2.1.4'
}
```

### Permissions Setup
```xml
<!-- AndroidManifest.xml -->
<uses-permission android:name="android.permission.BLUETOOTH" />
<uses-permission android:name="android.permission.BLUETOOTH_ADMIN" />
<uses-permission android:name="android.permission.BLUETOOTH_CONNECT" />
<uses-permission android:name="android.permission.BLUETOOTH_ADVERTISE" />
<uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />

<!-- LE Audio specific permissions -->
<uses-permission android:name="android.permission.BLUETOOTH_SCAN" />
<uses-permission android:name="android.permission.RECORD_AUDIO" />

<uses-feature 
    android:name="android.hardware.bluetooth_le" 
    android:required="true" />
```

## Core Bluetooth Implementation

### 1. Bluetooth Manager Class
```kotlin
import android.bluetooth.*
import android.bluetooth.le.*
import android.content.Context
import android.media.AudioManager
import android.media.MediaPlayer
import kotlinx.coroutines.*

class BluetoothAudioManager(private val context: Context) {
    private val bluetoothManager: BluetoothManager by lazy {
        context.getSystemService(Context.BLUETOOTH_SERVICE) as BluetoothManager
    }
    
    private val bluetoothAdapter: BluetoothAdapter by lazy {
        bluetoothManager.adapter
    }
    
    private val audioManager: AudioManager by lazy {
        context.getSystemService(Context.AUDIO_SERVICE) as AudioManager
    }
    
    private val connectedDevices = mutableListOf<BluetoothDevice>()
    private val deviceVolumeMap = mutableMapOf<String, Int>()
    
    companion object {
        private const val TAG = "BluetoothAudioManager"
    }
}
```

### 2. Device Discovery and Connection
```kotlin
class BluetoothAudioManager {
    private val bluetoothLeScanner: BluetoothLeScanner by lazy {
        bluetoothAdapter.bluetoothLeScanner
    }
    
    private val scanCallback = object : ScanCallback() {
        override fun onScanResult(callbackType: Int, result: ScanResult) {
            val device = result.device
            if (isAudioDevice(device)) {
                onDeviceFound(device)
            }
        }
        
        override fun onScanFailed(errorCode: Int) {
            Log.e(TAG, "Scan failed with error: $errorCode")
        }
    }
    
    fun startScanning() {
        if (!bluetoothAdapter.isEnabled) {
            // Request Bluetooth enable
            return
        }
        
        val scanSettings = ScanSettings.Builder()
            .setScanMode(ScanSettings.SCAN_MODE_LOW_LATENCY)
            .build()
            
        val scanFilters = listOf(
            ScanFilter.Builder()
                .setServiceUuid(ParcelUuid.fromString("0000110A-0000-1000-8000-00805F9B34FB")) // A2DP
                .build()
        )
        
        bluetoothLeScanner.startScan(scanFilters, scanSettings, scanCallback)
    }
    
    fun connectToDevice(device: BluetoothDevice) {
        val gatt = device.connectGatt(context, false, gattCallback)
        // Store GATT connection for management
    }
    
    private val gattCallback = object : BluetoothGattCallback() {
        override fun onConnectionStateChange(gatt: BluetoothGatt?, status: Int, newState: Int) {
            when (newState) {
                BluetoothProfile.STATE_CONNECTED -> {
                    connectedDevices.add(gatt?.device!!)
                    gatt.discoverServices()
                }
                BluetoothProfile.STATE_DISCONNECTED -> {
                    connectedDevices.remove(gatt?.device)
                }
            }
        }
        
        override fun onServicesDiscovered(gatt: BluetoothGatt?, status: Int) {
            if (status == BluetoothGatt.GATT_SUCCESS) {
                setupAudioService(gatt)
            }
        }
    }
}
```

### 3. LE Audio Implementation
```kotlin
class LEAudioManager(private val context: Context) {
    private var leAudioBroadcast: BluetoothLeBroadcast? = null
    
    fun startBroadcast() {
        val bluetoothManager = context.getSystemService(Context.BLUETOOTH_SERVICE) as BluetoothManager
        val adapter = bluetoothManager.adapter
        
        // Request LE Audio broadcast capability
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU) {
            leAudioBroadcast = adapter.getProfileProxy(
                context,
                object : BluetoothProfile.ServiceListener {
                    override fun onServiceConnected(profile: Int, proxy: BluetoothProfile?) {
                        if (proxy is BluetoothLeBroadcast) {
                            setupBroadcast(proxy)
                        }
                    }
                    
                    override fun onServiceDisconnected(profile: Int) {}
                },
                BluetoothProfile.LE_AUDIO_BROADCAST
            ) as? BluetoothLeBroadcast
        }
    }
    
    private fun setupBroadcast(broadcast: BluetoothLeBroadcast) {
        // Configure LE Audio broadcast parameters
        val broadcastSettings = BluetoothLeBroadcastSettings.Builder()
            .setCodec(BluetoothCodecConfig.SOURCE_CODEC_TYPE_LC3)
            .build()
            
        broadcast.startBroadcast(broadcastSettings)
    }
}
```

### 4. Volume Control Implementation
```kotlin
class VolumeController(private val audioManager: AudioManager) {
    private val deviceVolumeControllers = mutableMapOf<String, VolumeControl>()
    
    fun setVolumeForDevice(deviceId: String, volume: Int) {
        deviceVolumeControllers[deviceId]?.setVolume(volume)
        
        // For LE Audio devices, use individual volume control
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU) {
            setLEAudioDeviceVolume(deviceId, volume)
        } else {
            // Fallback for traditional Bluetooth
            setBluetoothClassicVolume(deviceId, volume)
        }
    }
    
    private fun setLEAudioDeviceVolume(deviceId: String, volume: Int) {
        // Use LE Audio volume control APIs
        val volumeControl = VolumeControl.Builder()
            .setVolume(volume)
            .setMute(false)
            .build()
            
        // Apply to specific device
    }
    
    private fun setBluetoothClassicVolume(deviceId: String, volume: Int) {
        // Traditional A2DP volume control
        audioManager.setStreamVolume(
            AudioManager.STREAM_MUSIC,
            volume,
            AudioManager.FLAG_SHOW_UI
        )
    }
    
    fun getMasterVolume(): Int {
        return audioManager.getStreamVolume(AudioManager.STREAM_MUSIC)
    }
    
    fun setMasterVolume(volume: Int) {
        audioManager.setStreamVolume(
            AudioManager.STREAM_MUSIC,
            volume,
            AudioManager.FLAG_SHOW_UI
        )
    }
}
```

### 5. Audio Streaming Implementation
```kotlin
class AudioStreamer {
    private var mediaPlayer: MediaPlayer? = null
    private val audioFocusRequest: AudioFocusRequest by lazy {
        AudioFocusRequest.Builder(AudioManager.AUDIOFOCUS_GAIN)
            .setAudioAttributes(
                AudioAttributes.Builder()
                    .setUsage(AudioAttributes.USAGE_MEDIA)
                    .setContentType(AudioAttributes.CONTENT_TYPE_MUSIC)
                    .build()
            )
            .build()
    }
    
    fun playAudioToAllDevices(audioFilePath: String) {
        requestAudioFocus()
        
        mediaPlayer = MediaPlayer().apply {
            setDataSource(audioFilePath)
            setAudioAttributes(
                AudioAttributes.Builder()
                    .setUsage(AudioAttributes.USAGE_MEDIA)
                    .setContentType(AudioAttributes.CONTENT_TYPE_MUSIC)
                    .build()
            )
            prepareAsync()
            setOnPreparedListener { start() }
        }
    }
    
    private fun requestAudioFocus() {
        val audioManager = context.getSystemService(Context.AUDIO_SERVICE) as AudioManager
        audioManager.requestAudioFocus(audioFocusRequest)
    }
}
```

## UI Implementation with Jetpack Compose

### 1. Main Activity
```kotlin
class MainActivity : ComponentActivity() {
    private lateinit var bluetoothManager: BluetoothAudioManager
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        
        bluetoothManager = BluetoothAudioManager(this)
        
        setContent {
            BluetoothAudioTheme {
                BluetoothAudioApp(bluetoothManager)
            }
        }
    }
}
```

### 2. Device List Composable
```kotlin
@Composable
fun DeviceListScreen(
    bluetoothManager: BluetoothAudioManager,
    modifier: Modifier = Modifier
) {
    val connectedDevices by bluetoothManager.connectedDevices.collectAsState()
    
    LazyColumn(modifier = modifier) {
        items(connectedDevices) { device ->
            DeviceItem(
                device = device,
                onVolumeChange = { volume ->
                    bluetoothManager.setVolumeForDevice(device.address, volume)
                },
                onDisconnect = {
                    bluetoothManager.disconnectDevice(device)
                }
            )
        }
    }
}

@Composable
fun DeviceItem(
    device: BluetoothDevice,
    onVolumeChange: (Int) -> Unit,
    onDisconnect: () -> Unit
) {
    var volume by remember { mutableStateOf(50) }
    
    Card(
        modifier = Modifier
            .fillMaxWidth()
            .padding(8.dp)
    ) {
        Column(
            modifier = Modifier.padding(16.dp)
        ) {
            Text(
                text = device.name ?: "Unknown Device",
                style = MaterialTheme.typography.h6
            )
            
            Row(
                modifier = Modifier.fillMaxWidth(),
                horizontalArrangement = Arrangement.SpaceBetween,
                verticalAlignment = Alignment.CenterVertically
            ) {
                Text("Volume")
                Slider(
                    value = volume.toFloat(),
                    onValueChange = { 
                        volume = it.toInt()
                        onVolumeChange(volume)
                    },
                    valueRange = 0f..100f,
                    modifier = Modifier.weight(1f)
                )
                Text("$volume%")
            }
            
            Button(
                onClick = onDisconnect,
                modifier = Modifier.align(Alignment.End)
            ) {
                Text("Disconnect")
            }
        }
    }
}
```

### 3. Audio Controls
```kotlin
@Composable
fun AudioControlsScreen(
    bluetoothManager: BluetoothAudioManager,
    modifier: Modifier = Modifier
) {
    var isPlaying by remember { mutableStateOf(false) }
    var masterVolume by remember { mutableStateOf(50) }
    
    Column(
        modifier = modifier.padding(16.dp),
        horizontalAlignment = Alignment.CenterHorizontally
    ) {
        // Master Volume Control
        Text("Master Volume", style = MaterialTheme.typography.h6)
        Slider(
            value = masterVolume.toFloat(),
            onValueChange = { 
                masterVolume = it.toInt()
                bluetoothManager.setMasterVolume(masterVolume)
            },
            valueRange = 0f..100f
        )
        
        Spacer(modifier = Modifier.height(32.dp))
        
        // Playback Controls
        Row(
            horizontalArrangement = Arrangement.SpaceEvenly,
            modifier = Modifier.fillMaxWidth()
        ) {
            IconButton(
                onClick = { bluetoothManager.previousTrack() }
            ) {
                Icon(Icons.Default.SkipPrevious, contentDescription = "Previous")
            }
            
            IconButton(
                onClick = { 
                    if (isPlaying) {
                        bluetoothManager.pauseAudio()
                    } else {
                        bluetoothManager.playAudio()
                    }
                    isPlaying = !isPlaying
                }
            ) {
                Icon(
                    if (isPlaying) Icons.Default.Pause else Icons.Default.PlayArrow,
                    contentDescription = if (isPlaying) "Pause" else "Play"
                )
            }
            
            IconButton(
                onClick = { bluetoothManager.nextTrack() }
            ) {
                Icon(Icons.Default.SkipNext, contentDescription = "Next")
            }
        }
    }
}
```

## Testing Strategy

### 1. Unit Tests
```kotlin
@RunWith(AndroidJUnit4::class)
class BluetoothAudioManagerTest {
    
    @Mock
    private lateinit var context: Context
    
    @Mock
    private lateinit var bluetoothManager: BluetoothManager
    
    private lateinit var audioManager: BluetoothAudioManager
    
    @Before
    fun setup() {
        MockitoAnnotations.openMocks(this)
        audioManager = BluetoothAudioManager(context)
    }
    
    @Test
    fun testDeviceConnection() {
        // Test device connection logic
    }
    
    @Test
    fun testVolumeControl() {
        // Test individual volume controls
    }
}
```

### 2. Integration Tests
```kotlin
@RunWith(AndroidJUnit4::class)
@LargeTest
class BluetoothIntegrationTest {
    
    @Test
    fun testMultipleDeviceConnection() {
        // Test connecting to multiple devices
    }
    
    @Test
    fun testAudioSynchronization() {
        // Test audio sync across devices
    }
}
```

## Performance Optimization

### Memory Management
```kotlin
class BluetoothAudioManager {
    override fun onDestroy() {
        connectedDevices.forEach { device ->
            disconnectDevice(device)
        }
        bluetoothLeScanner.stopScan(scanCallback)
        mediaPlayer?.release()
    }
}
```

### Battery Optimization
- Use efficient scan modes
- Implement connection pooling
- Optimize audio codec settings
- Use background processing judiciously

## Deployment Configuration

### ProGuard Rules
```proguard
# app/proguard-rules.pro
-keep class android.bluetooth.** { *; }
-keep class androidx.core.** { *; }
-dontwarn android.bluetooth.**
```

### Build Variants
```gradle
android {
    buildTypes {
        debug {
            minifyEnabled false
            debuggable true
        }
        release {
            minifyEnabled true
            proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 'proguard-rules.pro'
        }
    }
}
```
