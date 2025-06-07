# React Native Implementation Guide - Bluetooth Audio Multicast

## Project Setup and Configuration

### Environment Requirements
- **Node.js 18+**
- **React Native CLI** or **Expo CLI**
- **Android Studio** (for Android development)
- **Xcode 14+** (for iOS development)
- **Physical devices** for Bluetooth testing

### Project Initialization
```bash
# Create new React Native project
npx react-native init BluetoothAudioApp
cd BluetoothAudioApp

# Or with Expo (will need to eject for Bluetooth)
npx create-expo-app BluetoothAudioApp --template
cd BluetoothAudioApp
expo eject
```

### Required Dependencies
```json
{
  "dependencies": {
    "react": "18.2.0",
    "react-native": "0.72.0",
    "react-native-ble-manager": "^10.1.0",
    "react-native-permissions": "^3.8.0",
    "react-native-sound": "^0.11.2",
    "react-native-slider": "^2.4.0",
    "react-native-vector-icons": "^9.2.0",
    "@react-native-async-storage/async-storage": "^1.19.0",
    "react-native-device-info": "^10.7.0"
  },
  "devDependencies": {
    "@types/react": "^18.2.0",
    "@types/react-native": "^0.72.0",
    "typescript": "^5.0.0"
  }
}
```

### Installation and Linking
```bash
# Install dependencies
npm install

# iOS specific setup
cd ios && pod install && cd ..

# Android permissions setup is automatic with React Native 0.60+
```

### Platform-Specific Configuration

#### iOS Configuration
```xml
<!-- ios/BluetoothAudioApp/Info.plist -->
<key>NSBluetoothAlwaysUsageDescription</key>
<string>This app uses Bluetooth to connect to multiple audio devices for synchronized playback</string>
<key>NSBluetoothPeripheralUsageDescription</key>
<string>This app needs Bluetooth access to manage audio device connections</string>
<key>NSMicrophoneUsageDescription</key>
<string>This app needs microphone access for audio processing</string>
```

#### Android Configuration
```xml
<!-- android/app/src/main/AndroidManifest.xml -->
<uses-permission android:name="android.permission.BLUETOOTH" />
<uses-permission android:name="android.permission.BLUETOOTH_ADMIN" />
<uses-permission android:name="android.permission.BLUETOOTH_CONNECT" />
<uses-permission android:name="android.permission.BLUETOOTH_SCAN" />
<uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />

<uses-feature android:name="android.hardware.bluetooth_le" android:required="true"/>
```

## Core Bluetooth Implementation

### 1. Bluetooth Manager Service
```typescript
// src/services/BluetoothManager.ts
import BleManager from 'react-native-ble-manager';
import {PermissionsAndroid, Platform, NativeEventEmitter, NativeModules} from 'react-native';

const BleManagerModule = NativeModules.BleManager;
const bleManagerEmitter = new NativeEventEmitter(BleManagerModule);

export interface BluetoothDevice {
  id: string;
  name: string;
  rssi: number;
  isConnected: boolean;
  volume: number;
}

class BluetoothAudioManager {
  private connectedDevices: Map<string, BluetoothDevice> = new Map();
  private scanning: boolean = false;
  private deviceDiscoveryListener?: any;
  private deviceConnectedListener?: any;
  private deviceDisconnectedListener?: any;

  constructor() {
    this.initializeBluetooth();
    this.setupEventListeners();
  }

  private async initializeBluetooth(): Promise<void> {
    try {
      await BleManager.start({showAlert: false});
      console.log('Bluetooth Manager initialized');
      
      if (Platform.OS === 'android') {
        await this.requestAndroidPermissions();
      }
    } catch (error) {
      console.error('Failed to initialize Bluetooth:', error);
    }
  }

  private async requestAndroidPermissions(): Promise<boolean> {
    try {
      const granted = await PermissionsAndroid.requestMultiple([
        PermissionsAndroid.PERMISSIONS.BLUETOOTH_SCAN,
        PermissionsAndroid.PERMISSIONS.BLUETOOTH_CONNECT,
        PermissionsAndroid.PERMISSIONS.ACCESS_FINE_LOCATION,
      ]);

      return (
        granted['android.permission.BLUETOOTH_SCAN'] === PermissionsAndroid.RESULTS.GRANTED &&
        granted['android.permission.BLUETOOTH_CONNECT'] === PermissionsAndroid.RESULTS.GRANTED &&
        granted['android.permission.ACCESS_FINE_LOCATION'] === PermissionsAndroid.RESULTS.GRANTED
      );
    } catch (error) {
      console.error('Permission request failed:', error);
      return false;
    }
  }

  private setupEventListeners(): void {
    this.deviceDiscoveryListener = bleManagerEmitter.addListener(
      'BleManagerDiscoverPeripheral',
      this.handleDeviceDiscovered.bind(this)
    );

    this.deviceConnectedListener = bleManagerEmitter.addListener(
      'BleManagerConnectPeripheral',
      this.handleDeviceConnected.bind(this)
    );

    this.deviceDisconnectedListener = bleManagerEmitter.addListener(
      'BleManagerDisconnectPeripheral',
      this.handleDeviceDisconnected.bind(this)
    );
  }

  public async startScanning(): Promise<void> {
    if (this.scanning) return;

    try {
      this.scanning = true;
      await BleManager.scan(['180F'], 10, true); // Battery service UUID as example
      console.log('Started scanning for devices');
    } catch (error) {
      console.error('Failed to start scanning:', error);
      this.scanning = false;
    }
  }

  public async stopScanning(): Promise<void> {
    if (!this.scanning) return;

    try {
      await BleManager.stopScan();
      this.scanning = false;
      console.log('Stopped scanning');
    } catch (error) {
      console.error('Failed to stop scanning:', error);
    }
  }

  public async connectToDevice(deviceId: string): Promise<boolean> {
    try {
      await BleManager.connect(deviceId);
      console.log(`Connected to device: ${deviceId}`);
      return true;
    } catch (error) {
      console.error(`Failed to connect to device ${deviceId}:`, error);
      return false;
    }
  }

  public async disconnectDevice(deviceId: string): Promise<boolean> {
    try {
      await BleManager.disconnect(deviceId);
      console.log(`Disconnected from device: ${deviceId}`);
      this.connectedDevices.delete(deviceId);
      return true;
    } catch (error) {
      console.error(`Failed to disconnect from device ${deviceId}:`, error);
      return false;
    }
  }

  public setDeviceVolume(deviceId: string, volume: number): void {
    const device = this.connectedDevices.get(deviceId);
    if (device) {
      device.volume = volume;
      this.connectedDevices.set(deviceId, device);
      // Implement actual volume control through Bluetooth
      this.sendVolumeCommand(deviceId, volume);
    }
  }

  private async sendVolumeCommand(deviceId: string, volume: number): Promise<void> {
    try {
      // This would need to be implemented based on the specific Bluetooth profile
      // For audio devices, this might involve writing to a specific characteristic
      const volumeCharacteristic = 'YOUR_VOLUME_CHARACTERISTIC_UUID';
      const volumeData = [Math.round(volume * 255 / 100)]; // Convert to 0-255 range
      
      await BleManager.write(deviceId, 'SERVICE_UUID', volumeCharacteristic, volumeData);
    } catch (error) {
      console.error('Failed to send volume command:', error);
    }
  }

  private handleDeviceDiscovered(device: any): void {
    if (this.isAudioDevice(device)) {
      const bluetoothDevice: BluetoothDevice = {
        id: device.id,
        name: device.name || 'Unknown Device',
        rssi: device.rssi,
        isConnected: false,
        volume: 50,
      };
      
      // Emit event or update state
      this.onDeviceDiscovered?.(bluetoothDevice);
    }
  }

  private handleDeviceConnected(device: any): void {
    const bluetoothDevice: BluetoothDevice = {
      id: device.peripheral,
      name: device.name || 'Unknown Device',
      rssi: 0,
      isConnected: true,
      volume: 50,
    };
    
    this.connectedDevices.set(device.peripheral, bluetoothDevice);
    this.onDeviceConnected?.(bluetoothDevice);
  }

  private handleDeviceDisconnected(device: any): void {
    this.connectedDevices.delete(device.peripheral);
    this.onDeviceDisconnected?.(device.peripheral);
  }

  private isAudioDevice(device: any): boolean {
    // Logic to determine if device is an audio device
    // Could check device name, services, etc.
    const audioKeywords = ['headphone', 'speaker', 'audio', 'airpod', 'beats'];
    const deviceName = (device.name || '').toLowerCase();
    return audioKeywords.some(keyword => deviceName.includes(keyword));
  }

  public getConnectedDevices(): BluetoothDevice[] {
    return Array.from(this.connectedDevices.values());
  }

  // Event callbacks (to be set by components)
  public onDeviceDiscovered?: (device: BluetoothDevice) => void;
  public onDeviceConnected?: (device: BluetoothDevice) => void;
  public onDeviceDisconnected?: (deviceId: string) => void;

  public cleanup(): void {
    this.deviceDiscoveryListener?.remove();
    this.deviceConnectedListener?.remove();
    this.deviceDisconnectedListener?.remove();
  }
}

export default new BluetoothAudioManager();
```

### 2. Audio Manager Service
```typescript
// src/services/AudioManager.ts
import Sound from 'react-native-sound';
import {Platform} from 'react-native';

export class AudioManager {
  private currentSound: Sound | null = null;
  private isPlaying: boolean = false;
  private masterVolume: number = 1.0;

  constructor() {
    Sound.setCategory('Playback');
  }

  public async loadAudio(audioUrl: string): Promise<boolean> {
    return new Promise((resolve) => {
      this.currentSound = new Sound(audioUrl, '', (error) => {
        if (error) {
          console.error('Failed to load audio:', error);
          resolve(false);
        } else {
          resolve(true);
        }
      });
    });
  }

  public play(): void {
    if (this.currentSound && !this.isPlaying) {
      this.currentSound.play((success) => {
        if (success) {
          console.log('Audio playback completed');
        } else {
          console.error('Audio playback failed');
        }
        this.isPlaying = false;
      });
      this.isPlaying = true;
    }
  }

  public pause(): void {
    if (this.currentSound && this.isPlaying) {
      this.currentSound.pause();
      this.isPlaying = false;
    }
  }

  public stop(): void {
    if (this.currentSound) {
      this.currentSound.stop();
      this.isPlaying = false;
    }
  }

  public setMasterVolume(volume: number): void {
    this.masterVolume = Math.max(0, Math.min(1, volume));
    if (this.currentSound) {
      this.currentSound.setVolume(this.masterVolume);
    }
  }

  public getMasterVolume(): number {
    return this.masterVolume;
  }

  public release(): void {
    if (this.currentSound) {
      this.currentSound.release();
      this.currentSound = null;
    }
  }
}

export default new AudioManager();
```

## React Native Components

### 1. Main App Component
```typescript
// App.tsx
import React, {useEffect, useState} from 'react';
import {
  SafeAreaView,
  StatusBar,
  StyleSheet,
  useColorScheme,
} from 'react-native';
import {Colors} from 'react-native/Libraries/NewAppScreen';

import BluetoothManager, {BluetoothDevice} from './src/services/BluetoothManager';
import AudioManager from './src/services/AudioManager';
import DeviceListScreen from './src/components/DeviceListScreen';
import AudioControlsScreen from './src/components/AudioControlsScreen';
import TabNavigator from './src/components/TabNavigator';

function App(): JSX.Element {
  const isDarkMode = useColorScheme() === 'dark';
  const [connectedDevices, setConnectedDevices] = useState<BluetoothDevice[]>([]);
  const [discoveredDevices, setDiscoveredDevices] = useState<BluetoothDevice[]>([]);

  useEffect(() => {
    // Setup Bluetooth event listeners
    BluetoothManager.onDeviceDiscovered = (device: BluetoothDevice) => {
      setDiscoveredDevices(prev => {
        const exists = prev.find(d => d.id === device.id);
        if (!exists) {
          return [...prev, device];
        }
        return prev;
      });
    };

    BluetoothManager.onDeviceConnected = (device: BluetoothDevice) => {
      setConnectedDevices(prev => [...prev, device]);
      setDiscoveredDevices(prev => prev.filter(d => d.id !== device.id));
    };

    BluetoothManager.onDeviceDisconnected = (deviceId: string) => {
      setConnectedDevices(prev => prev.filter(d => d.id !== deviceId));
    };

    return () => {
      BluetoothManager.cleanup();
      AudioManager.release();
    };
  }, []);

  const backgroundStyle = {
    backgroundColor: isDarkMode ? Colors.darker : Colors.lighter,
    flex: 1,
  };

  return (
    <SafeAreaView style={backgroundStyle}>
      <StatusBar
        barStyle={isDarkMode ? 'light-content' : 'dark-content'}
        backgroundColor={backgroundStyle.backgroundColor}
      />
      <TabNavigator
        connectedDevices={connectedDevices}
        discoveredDevices={discoveredDevices}
        onConnectDevice={BluetoothManager.connectToDevice.bind(BluetoothManager)}
        onDisconnectDevice={BluetoothManager.disconnectDevice.bind(BluetoothManager)}
        onStartScanning={BluetoothManager.startScanning.bind(BluetoothManager)}
        onStopScanning={BluetoothManager.stopScanning.bind(BluetoothManager)}
        onVolumeChange={BluetoothManager.setDeviceVolume.bind(BluetoothManager)}
      />
    </SafeAreaView>
  );
}

export default App;
```

### 2. Device List Component
```typescript
// src/components/DeviceListScreen.tsx
import React from 'react';
import {
  View,
  Text,
  FlatList,
  TouchableOpacity,
  StyleSheet,
} from 'react-native';
import Slider from '@react-native-community/slider';
import {BluetoothDevice} from '../services/BluetoothManager';

interface Props {
  connectedDevices: BluetoothDevice[];
  discoveredDevices: BluetoothDevice[];
  onConnectDevice: (deviceId: string) => void;
  onDisconnectDevice: (deviceId: string) => void;
  onVolumeChange: (deviceId: string, volume: number) => void;
  onStartScanning: () => void;
  onStopScanning: () => void;
}

const DeviceListScreen: React.FC<Props> = ({
  connectedDevices,
  discoveredDevices,
  onConnectDevice,
  onDisconnectDevice,
  onVolumeChange,
  onStartScanning,
  onStopScanning,
}) => {
  const renderConnectedDevice = ({item}: {item: BluetoothDevice}) => (
    <View style={styles.deviceItem}>
      <View style={styles.deviceInfo}>
        <Text style={styles.deviceName}>{item.name}</Text>
        <Text style={styles.deviceId}>ID: {item.id}</Text>
      </View>
      
      <View style={styles.volumeControl}>
        <Text style={styles.volumeLabel}>Volume: {Math.round(item.volume)}%</Text>
        <Slider
          style={styles.slider}
          minimumValue={0}
          maximumValue={100}
          value={item.volume}
          onValueChange={(value) => onVolumeChange(item.id, value)}
          minimumTrackTintColor="#1976D2"
          maximumTrackTintColor="#E0E0E0"
          thumbStyle={styles.sliderThumb}
        />
      </View>
      
      <TouchableOpacity
        style={styles.disconnectButton}
        onPress={() => onDisconnectDevice(item.id)}>
        <Text style={styles.buttonText}>Disconnect</Text>
      </TouchableOpacity>
    </View>
  );

  const renderDiscoveredDevice = ({item}: {item: BluetoothDevice}) => (
    <View style={styles.deviceItem}>
      <View style={styles.deviceInfo}>
        <Text style={styles.deviceName}>{item.name}</Text>
        <Text style={styles.deviceId}>ID: {item.id}</Text>
        <Text style={styles.rssi}>RSSI: {item.rssi} dBm</Text>
      </View>
      
      <TouchableOpacity
        style={styles.connectButton}
        onPress={() => onConnectDevice(item.id)}>
        <Text style={styles.buttonText}>Connect</Text>
      </TouchableOpacity>
    </View>
  );

  return (
    <View style={styles.container}>
      <View style={styles.section}>
        <Text style={styles.sectionTitle}>Connected Devices</Text>
        {connectedDevices.length === 0 ? (
          <Text style={styles.emptyText}>No devices connected</Text>
        ) : (
          <FlatList
            data={connectedDevices}
            renderItem={renderConnectedDevice}
            keyExtractor={(item) => item.id}
            style={styles.list}
          />
        )}
      </View>

      <View style={styles.section}>
        <View style={styles.sectionHeader}>
          <Text style={styles.sectionTitle}>Available Devices</Text>
          <View style={styles.scanButtons}>
            <TouchableOpacity style={styles.scanButton} onPress={onStartScanning}>
              <Text style={styles.buttonText}>Scan</Text>
            </TouchableOpacity>
            <TouchableOpacity style={styles.scanButton} onPress={onStopScanning}>
              <Text style={styles.buttonText}>Stop</Text>
            </TouchableOpacity>
          </View>
        </View>
        
        {discoveredDevices.length === 0 ? (
          <Text style={styles.emptyText}>No devices found</Text>
        ) : (
          <FlatList
            data={discoveredDevices}
            renderItem={renderDiscoveredDevice}
            keyExtractor={(item) => item.id}
            style={styles.list}
          />
        )}
      </View>
    </View>
  );
};

const styles = StyleSheet.create({
  container: {
    flex: 1,
    padding: 16,
    backgroundColor: '#F5F5F5',
  },
  section: {
    flex: 1,
    marginBottom: 16,
  },
  sectionHeader: {
    flexDirection: 'row',
    justifyContent: 'space-between',
    alignItems: 'center',
    marginBottom: 12,
  },
  sectionTitle: {
    fontSize: 18,
    fontWeight: 'bold',
    color: '#333',
  },
  scanButtons: {
    flexDirection: 'row',
    gap: 8,
  },
  scanButton: {
    backgroundColor: '#1976D2',
    paddingHorizontal: 12,
    paddingVertical: 6,
    borderRadius: 4,
  },
  deviceItem: {
    backgroundColor: 'white',
    padding: 12,
    marginBottom: 8,
    borderRadius: 8,
    elevation: 2,
    shadowColor: '#000',
    shadowOffset: {width: 0, height: 1},
    shadowOpacity: 0.2,
    shadowRadius: 2,
  },
  deviceInfo: {
    marginBottom: 8,
  },
  deviceName: {
    fontSize: 16,
    fontWeight: 'bold',
    color: '#333',
  },
  deviceId: {
    fontSize: 12,
    color: '#666',
    marginTop: 2,
  },
  rssi: {
    fontSize: 12,
    color: '#666',
    marginTop: 2,
  },
  volumeControl: {
    marginVertical: 8,
  },
  volumeLabel: {
    fontSize: 14,
    color: '#333',
    marginBottom: 4,
  },
  slider: {
    width: '100%',
    height: 40,
  },
  sliderThumb: {
    backgroundColor: '#1976D2',
  },
  connectButton: {
    backgroundColor: '#4CAF50',
    paddingVertical: 8,
    paddingHorizontal: 16,
    borderRadius: 4,
    alignSelf: 'flex-end',
  },
  disconnectButton: {
    backgroundColor: '#F44336',
    paddingVertical: 8,
    paddingHorizontal: 16,
    borderRadius: 4,
    alignSelf: 'flex-end',
  },
  buttonText: {
    color: 'white',
    fontWeight: 'bold',
    fontSize: 14,
  },
  list: {
    flex: 1,
  },
  emptyText: {
    textAlign: 'center',
    color: '#666',
    fontStyle: 'italic',
    marginTop: 16,
  },
});

export default DeviceListScreen;
```

### 3. Audio Controls Component
```typescript
// src/components/AudioControlsScreen.tsx
import React, {useState, useEffect} from 'react';
import {
  View,
  Text,
  TouchableOpacity,
  StyleSheet,
  Alert,
} from 'react-native';
import Slider from '@react-native-community/slider';
import AudioManager from '../services/AudioManager';
import {BluetoothDevice} from '../services/BluetoothManager';

interface Props {
  connectedDevices: BluetoothDevice[];
}

const AudioControlsScreen: React.FC<Props> = ({connectedDevices}) => {
  const [isPlaying, setIsPlaying] = useState(false);
  const [masterVolume, setMasterVolume] = useState(50);
  const [currentTrack, setCurrentTrack] = useState('No track loaded');

  useEffect(() => {
    // Load a sample audio file
    loadSampleAudio();
  }, []);

  const loadSampleAudio = async () => {
    const audioUrl = 'https://www.soundjay.com/misc/sounds/bell-ringing-05.mp3';
    const loaded = await AudioManager.loadAudio(audioUrl);
    if (loaded) {
      setCurrentTrack('Sample Audio Track');
    } else {
      Alert.alert('Error', 'Failed to load audio file');
    }
  };

  const handlePlayPause = () => {
    if (connectedDevices.length === 0) {
      Alert.alert('No Devices', 'Please connect at least one audio device first');
      return;
    }

    if (isPlaying) {
      AudioManager.pause();
      setIsPlaying(false);
    } else {
      AudioManager.play();
      setIsPlaying(true);
    }
  };

  const handleStop = () => {
    AudioManager.stop();
    setIsPlaying(false);
  };

  const handleMasterVolumeChange = (volume: number) => {
    setMasterVolume(volume);
    AudioManager.setMasterVolume(volume / 100);
  };

  const getConnectionStatus = () => {
    const count = connectedDevices.length;
    if (count === 0) return 'No devices connected';
    if (count === 1) return '1 device connected';
    return `${count} devices connected`;
  };

  return (
    <View style={styles.container}>
      <View style={styles.statusSection}>
        <Text style={styles.statusTitle}>Connection Status</Text>
        <Text style={styles.statusText}>{getConnectionStatus()}</Text>
        <Text style={styles.trackTitle}>Current Track</Text>
        <Text style={styles.trackName}>{currentTrack}</Text>
      </View>

      <View style={styles.volumeSection}>
        <Text style={styles.volumeTitle}>Master Volume</Text>
        <Text style={styles.volumeValue}>{Math.round(masterVolume)}%</Text>
        <Slider
          style={styles.masterSlider}
          minimumValue={0}
          maximumValue={100}
          value={masterVolume}
          onValueChange={handleMasterVolumeChange}
          minimumTrackTintColor="#1976D2"
          maximumTrackTintColor="#E0E0E0"
          thumbStyle={styles.sliderThumb}
        />
      </View>

      <View style={styles.controlsSection}>
        <Text style={styles.controlsTitle}>Playback Controls</Text>
        <View style={styles.buttonRow}>
          <TouchableOpacity
            style={styles.controlButton}
            onPress={() => {/* Previous track */}}>
            <Text style={styles.controlButtonText}>⏮</Text>
          </TouchableOpacity>

          <TouchableOpacity
            style={[styles.controlButton, styles.playButton]}
            onPress={handlePlayPause}>
            <Text style={styles.controlButtonText}>
              {isPlaying ? '⏸' : '▶'}
            </Text>
          </TouchableOpacity>

          <TouchableOpacity
            style={styles.controlButton}
            onPress={handleStop}>
            <Text style={styles.controlButtonText}>⏹</Text>
          </TouchableOpacity>

          <TouchableOpacity
            style={styles.controlButton}
            onPress={() => {/* Next track */}}>
            <Text style={styles.controlButtonText}>⏭</Text>
          </TouchableOpacity>
        </View>
      </View>

      <View style={styles.devicesSection}>
        <Text style={styles.devicesTitle}>Connected Devices</Text>
        {connectedDevices.length === 0 ? (
          <Text style={styles.noDevicesText}>
            Connect audio devices from the Devices tab to start streaming
          </Text>
        ) : (
          connectedDevices.map((device) => (
            <View key={device.id} style={styles.deviceStatus}>
              <Text style={styles.deviceStatusName}>{device.name}</Text>
              <Text style={styles.deviceStatusVolume}>
                Volume: {Math.round(device.volume)}%
              </Text>
            </View>
          ))
        )}
      </View>
    </View>
  );
};

const styles = StyleSheet.create({
  container: {
    flex: 1,
    padding: 16,
    backgroundColor: '#F5F5F5',
  },
  statusSection: {
    backgroundColor: 'white',
    padding: 16,
    borderRadius: 8,
    marginBottom: 16,
    elevation: 2,
    shadowColor: '#000',
    shadowOffset: {width: 0, height: 1},
    shadowOpacity: 0.2,
    shadowRadius: 2,
  },
  statusTitle: {
    fontSize: 16,
    fontWeight: 'bold',
    color: '#333',
    marginBottom: 8,
  },
  statusText: {
    fontSize: 14,
    color: '#666',
    marginBottom: 12,
  },
  trackTitle: {
    fontSize: 16,
    fontWeight: 'bold',
    color: '#333',
    marginBottom: 8,
  },
  trackName: {
    fontSize: 14,
    color: '#666',
  },
  volumeSection: {
    backgroundColor: 'white',
    padding: 16,
    borderRadius: 8,
    marginBottom: 16,
    elevation: 2,
    shadowColor: '#000',
    shadowOffset: {width: 0, height: 1},
    shadowOpacity: 0.2,
    shadowRadius: 2,
  },
  volumeTitle: {
    fontSize: 16,
    fontWeight: 'bold',
    color: '#333',
    marginBottom: 8,
  },
  volumeValue: {
    fontSize: 14,
    color: '#666',
    marginBottom: 8,
  },
  masterSlider: {
    width: '100%',
    height: 40,
  },
  sliderThumb: {
    backgroundColor: '#1976D2',
  },
  controlsSection: {
    backgroundColor: 'white',
    padding: 16,
    borderRadius: 8,
    marginBottom: 16,
    elevation: 2,
    shadowColor: '#000',
    shadowOffset: {width: 0, height: 1},
    shadowOpacity: 0.2,
    shadowRadius: 2,
  },
  controlsTitle: {
    fontSize: 16,
    fontWeight: 'bold',
    color: '#333',
    marginBottom: 16,
    textAlign: 'center',
  },
  buttonRow: {
    flexDirection: 'row',
    justifyContent: 'center',
    alignItems: 'center',
    gap: 16,
  },
  controlButton: {
    backgroundColor: '#1976D2',
    width: 50,
    height: 50,
    borderRadius: 25,
    justifyContent: 'center',
    alignItems: 'center',
  },
  playButton: {
    backgroundColor: '#4CAF50',
    width: 60,
    height: 60,
    borderRadius: 30,
  },
  controlButtonText: {
    color: 'white',
    fontSize: 18,
    fontWeight: 'bold',
  },
  devicesSection: {
    backgroundColor: 'white',
    padding: 16,
    borderRadius: 8,
    elevation: 2,
    shadowColor: '#000',
    shadowOffset: {width: 0, height: 1},
    shadowOpacity: 0.2,
    shadowRadius: 2,
  },
  devicesTitle: {
    fontSize: 16,
    fontWeight: 'bold',
    color: '#333',
    marginBottom: 12,
  },
  noDevicesText: {
    fontSize: 14,
    color: '#666',
    textAlign: 'center',
    fontStyle: 'italic',
  },
  deviceStatus: {
    flexDirection: 'row',
    justifyContent: 'space-between',
    alignItems: 'center',
    paddingVertical: 8,
    borderBottomWidth: 1,
    borderBottomColor: '#E0E0E0',
  },
  deviceStatusName: {
    fontSize: 14,
    color: '#333',
    fontWeight: '500',
  },
  deviceStatusVolume: {
    fontSize: 12,
    color: '#666',
  },
});

export default AudioControlsScreen;
```

## Testing and Deployment

### Testing Strategy
1. **Unit Tests**: Test Bluetooth manager functions
2. **Integration Tests**: Test with actual Bluetooth devices
3. **Device Compatibility**: Test across various device models
4. **Performance Tests**: Battery usage and audio latency testing

### Build and Deploy
```bash
# Android build
npx react-native run-android --variant=release

# iOS build
npx react-native run-ios --configuration Release

# Generate APK/AAB
cd android && ./gradlew assembleRelease

# iOS Archive
open ios/BluetoothAudioApp.xcworkspace
```

## Limitations and Considerations

### Current Limitations
- Web Bluetooth API restrictions in React Native
- Platform-specific Bluetooth audio profiles
- Limited simultaneous connection support
- Battery optimization challenges

### Future Enhancements
- LE Audio multicast integration
- Advanced audio codecs support
- Cross-platform audio synchronization
- Enhanced device management features

This comprehensive guide provides the foundation for implementing Bluetooth audio multicast functionality in React Native, with platform-specific optimizations and real-world considerations.
