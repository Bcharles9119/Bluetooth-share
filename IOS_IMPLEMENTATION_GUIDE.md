# iOS Implementation Guide - Bluetooth Audio Multicast

## Development Environment Setup

### Requirements
- **Xcode 14+**
- **iOS 15+ target** (for LE Audio support)
- **Physical devices** for Bluetooth testing
- **Apple Developer Account** for device provisioning

### Project Configuration
```swift
// Info.plist additions
<key>NSBluetoothAlwaysUsageDescription</key>
<string>This app uses Bluetooth to connect to multiple audio devices</string>
<key>NSBluetoothPeripheralUsageDescription</key>
<string>This app broadcasts audio to multiple Bluetooth devices</string>
```

## Core Bluetooth Implementation

### 1. Bluetooth Manager Setup
```swift
import CoreBluetooth
import AVFoundation

class BluetoothAudioManager: NSObject {
    private var centralManager: CBCentralManager!
    private var peripheralManager: CBPeripheralManager!
    private var connectedDevices: [CBPeripheral] = []
    private var audioEngine: AVAudioEngine!
    
    override init() {
        super.init()
        setupBluetoothManagers()
        setupAudioEngine()
    }
    
    private func setupBluetoothManagers() {
        centralManager = CBCentralManager(delegate: self, queue: nil)
        peripheralManager = CBPeripheralManager(delegate: self, queue: nil)
    }
}
```

### 2. Device Discovery and Connection
```swift
extension BluetoothAudioManager: CBCentralManagerDelegate {
    func centralManagerDidUpdateState(_ central: CBCentralManager) {
        if central.state == .poweredOn {
            startScanning()
        }
    }
    
    func startScanning() {
        // Scan for audio devices with A2DP or LE Audio support
        centralManager.scanForPeripherals(
            withServices: [CBUUID(string: "1811")], // Audio Input Control Service
            options: [CBCentralManagerScanOptionAllowDuplicatesKey: false]
        )
    }
    
    func connectToDevice(_ peripheral: CBPeripheral) {
        centralManager.connect(peripheral, options: nil)
    }
    
    func centralManager(_ central: CBCentralManager, didConnect peripheral: CBPeripheral) {
        connectedDevices.append(peripheral)
        peripheral.delegate = self
        peripheral.discoverServices(nil)
        
        // Setup audio streaming for this device
        setupAudioStreamForDevice(peripheral)
    }
}
```

### 3. LE Audio Multicast Implementation
```swift
class LEAudioBroadcaster {
    private var broadcaster: AVAudioSession!
    
    func startBroadcast() {
        do {
            try AVAudioSession.sharedInstance().setCategory(
                .playback,
                mode: .default,
                options: [.allowBluetooth, .allowBluetoothA2DP]
            )
            try AVAudioSession.sharedInstance().setActive(true)
            
            // Configure for LE Audio broadcast
            setupLEAudioBroadcast()
        } catch {
            print("Audio session setup failed: \(error)")
        }
    }
    
    private func setupLEAudioBroadcast() {
        // Implementation for LE Audio multicast
        // This will use iOS 16+ APIs when available
    }
}
```

### 4. Volume Control Implementation
```swift
class VolumeController {
    private var deviceVolumes: [String: Float] = [:]
    
    func setVolumeForDevice(_ deviceId: String, volume: Float) {
        deviceVolumes[deviceId] = volume
        applyVolumeToDevice(deviceId, volume: volume)
    }
    
    private func applyVolumeToDevice(_ deviceId: String, volume: Float) {
        // Apply volume control through audio mixer
        if let audioNode = getAudioNodeForDevice(deviceId) {
            audioNode.volume = volume
        }
    }
}
```

### 5. Audio Engine Setup
```swift
extension BluetoothAudioManager {
    private func setupAudioEngine() {
        audioEngine = AVAudioEngine()
        
        let playerNode = AVAudioPlayerNode()
        let mixer = audioEngine.mainMixerNode
        
        audioEngine.attach(playerNode)
        audioEngine.connect(playerNode, to: mixer, format: nil)
        
        do {
            try audioEngine.start()
        } catch {
            print("Audio engine start failed: \(error)")
        }
    }
    
    func playAudioToAllDevices(_ audioFile: AVAudioFile) {
        for device in connectedDevices {
            playAudioToDevice(device, audioFile: audioFile)
        }
    }
}
```

## UI Implementation with SwiftUI

### 1. Main View Controller
```swift
import SwiftUI

struct ContentView: View {
    @StateObject private var bluetoothManager = BluetoothAudioManager()
    
    var body: some View {
        NavigationView {
            VStack {
                DeviceListView(manager: bluetoothManager)
                AudioControlsView(manager: bluetoothManager)
            }
            .navigationTitle("Bluetooth Audio")
        }
    }
}
```

### 2. Device Management View
```swift
struct DeviceListView: View {
    @ObservedObject var manager: BluetoothAudioManager
    
    var body: some View {
        List {
            ForEach(manager.connectedDevices, id: \.identifier) { device in
                DeviceRowView(device: device, manager: manager)
            }
        }
        .navigationTitle("Connected Devices")
    }
}

struct DeviceRowView: View {
    let device: CBPeripheral
    @ObservedObject var manager: BluetoothAudioManager
    @State private var volume: Float = 0.5
    
    var body: some View {
        HStack {
            Text(device.name ?? "Unknown Device")
            Spacer()
            VStack {
                Text("Volume")
                Slider(value: $volume, in: 0...1) { _ in
                    manager.setVolumeForDevice(device.identifier.uuidString, volume: volume)
                }
            }
            Button("Disconnect") {
                manager.disconnectDevice(device)
            }
        }
    }
}
```

## Testing Strategy

### 1. Unit Tests
```swift
import XCTest
@testable import BluetoothAudio

class BluetoothManagerTests: XCTestCase {
    var manager: BluetoothAudioManager!
    
    override func setUp() {
        manager = BluetoothAudioManager()
    }
    
    func testDeviceConnection() {
        // Test device connection logic
    }
    
    func testVolumeControl() {
        // Test individual volume controls
    }
}
```

### 2. Integration Testing
- Test with multiple physical Bluetooth devices
- Verify audio synchronization across devices
- Test connection stability during extended use
- Verify volume control independence

## Deployment Considerations

### App Store Requirements
- Privacy policy for Bluetooth usage
- Clear user permissions explanation
- Device compatibility testing
- Performance optimization documentation

### Code Signing
```bash
# Development provisioning profile
DEVELOPMENT_TEAM = "YOUR_TEAM_ID"
CODE_SIGN_IDENTITY = "iPhone Developer"

# Distribution profile
CODE_SIGN_IDENTITY = "iPhone Distribution"
```

### Performance Optimization
- Implement connection pooling for multiple devices
- Use background processing for audio streaming
- Optimize battery usage with efficient Bluetooth protocols
- Implement automatic reconnection logic
