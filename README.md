# Mentalab Explore Core SDK

A Flutter plugin for interfacing with Mentalab Explore devices. This package provides a simple API to connect to Explore devices, process and record biosignal data, measure impedance, configure, and apply digital filters.

> ⚠️ **Note:** This package only supports Bluetooth Low Energy (BLE) connections. Bluetooth Classic is not supported.

## Features

- 🔌 Bluetooth connection management for Explore devices
- 📈 ExG data processing
- 🎛️ Impedance measurement
- ⚡ Sampling rate configuration
- 📊 Digital filtering with adjustable parameters
- 💾 Data recording capabilities
- 🛠️ Simple and intuitive API

## Getting started

To use this plugin, add `explore_flutter_core` as a dependency in your `pubspec.yaml` file.

```yaml
dependencies:
  explore_flutter_core:
    git:
      url: https://github.com/Mentalab-hub/explore-flutter-core.git
      ref: v0.0.1
```

Then import the package in your Dart file:

```dart
import 'package:explore_flutter_core/explore_flutter_core.dart';
```

The plugin automatically includes all necessary dependencies, including [`flutter_blue_plus`](https://github.com/boskokg/flutter_blue_plus).

## Usage
For comprehensive examples, please refer to the [examples folder](./example/).

### Initialization & Connection

This section demonstrates how to use `FlutterBluePlus` to scan for available Explore devices and connect to the first found device. The `BluetoothDevice` obtained from the scan is then passed to the `ExploreDevice.setBleDevice` method.

```dart
import 'package:flutter_blue_plus/flutter_blue_plus.dart';
import 'package:explore_flutter_core/explore_flutter_core.dart';

List<ScanResult> _devicesList = [];
bool connecting = false;

// First, scan for your Explore device using [flutter_blue_plus]
await FlutterBluePlus.startScan(timeout: const Duration(seconds: 4));

// Listen for scan results
FlutterBluePlus.scanResults.listen((results) async {
    // Filter for Explore devices
    _devicesList = results
        .where((result) =>
            result.device.advName.toLowerCase().contains('explore'))
        .toList();

    // Automatically connect to the first found device
    if (_devicesList.isNotEmpty && !connecting) {
      ScanResult exploreDeviceScanResult = _devicesList.first;
      connecting = true;
      // Connect to the device
        await exploreDeviceScanResult.device.connect().timeout(
          const Duration(seconds: 5),
          onTimeout: () {
            throw TimeoutException('Timeout');
          },
        );
        ExploreDevice _exploreDevice = ExploreDevice();
        await _exploreDevice.setBleDevice(exploreDeviceScanResult.device);
    }
});
```

> [!NOTE]
> For more information on Bluetooth connection with FlutterBluePlus, please refer to the [flutter_blue_plus documentation](https://pub.dev/packages/flutter_blue_plus).

#### Disconnect

```dart
// Disconnect from the device
await ExploreDevice.disconnect();
```

### Enable Impedance Measurement Mode

To get impedance data, you need to first create and register an `ImpedanceCalculationSubscriber` to collect raw impedance data and calculate impedance values. This subscriber must be registered before sending the enable impedance command to ensure accurate data collection. After enabling impedance, you should create and register an `ImpedanceSubscriber` to collect the impedance data.

```dart
// Create & register Impedance Calculation Subscriber - collects raw data and publishes impedance data
ImpedanceCalculationSubscriber impedanceCalculationSubscriber = 
  ImpedanceCalculationSubscriber(exploreDevice.channelCount.value);
ContentServer.instance.registerSubscriber(impedanceCalculationSubscriber);

// Enable impedance - send enable impedance command
await _exploreDevice.sendImpedanceEnableCommand();

// Create & register Impedance Subscriber - collects impedance data
ImpedanceSubscriber impedanceSubscriber = ImpedanceSubscriber();
ContentServer.instance.registerSubscriber(impedanceSubscriber);
```

> [!WARNING]
> ImpedanceCalculationSubscriber must be registered before sending the enable impedance command.
> Otherwise, the impedance values may not be accurate.

### Disable Impedance Measurement Mode - (ExG Mode)

#### Disable Impedance Measurement Mode
```dart
// Disable impedance - send disable impedance command
await _exploreDevice.sendImpedanceDisableCommand();

// Deregister ImpedanceCalculationSubscriber & impedanceSubscriber
ContentServer.instance.deRegisterSubscriber(impedanceCalculationSubscriber);
ContentServer.instance.deRegisterSubscriber(impedanceSubscriber);
```

#### Switch to ExG Mode

```dart
// Create & register RawExgSubscriber - collects raw ExG data
RawExgSubscriber rawExgSubscriber = RawExgSubscriber(_exploreDevice.channelCount.value, 
  _exploreDevice.samplingRate.integerRepresentation.toDouble());
ContentServer.instance.registerSubscriber(rawExgSubscriber);

// Create & register ExgSubscriber - collects filtered ExG data
// Named parameter downSampleRate defaults to 2
ExgSubscriber exgSubscriber = ExgSubscriber(_exploreDevice.channelCount.value, downSampleRate: 2);
ContentServer.instance.registerSubscriber(exgSubscriber);
```

> [!WARNING]
> RawExgSubscriber and ImpedanceCalculationSubscriber are subscribed to the same topic. This means that you will need to deregister the ImpedanceCalculationSubscriber before deregistering the RawExgSubscriber or vice versa.

### Configure Sampling Rate

```dart
await exploreDevice.setSPS(sps);
```

> [!NOTE]
> Changing the sampling rate is not communicated across all subscribers, therefore you may need to update the sampling rate values in needed subscribers. E.g., you may need to [adjust the filters](#adjust-filter-parameters) after changing the sampling rate.

### Record Data

```dart
// Instantiate RecordTask, set duration, and call
RecordTask recordTask = RecordTask(filename, _exploreDevice.samplingRate.integerRepresentation, exploreDevice.channelCount.value);
recordTask.setDuration(duration);
_recordTask.call();

// Stop recording
recordTask.close();
```

> [!NOTE]
> Recorded files are saved in the application's documents directory for iOS, and in the download directory for Android.

### Adjust Filter Parameters

```dart
// Adjust filter parameters
rawExgSubscriber.adjustFilters(FilterParameters(highCutoffFreq, lowCutoffFreq, notchFreq));
```

## Package Structure Overview

The `explore_flutter_core` package is designed to facilitate communication with Mentalab Explore Pro devices. It provides a flexible architecture that allows developers to extend its functionality by creating custom subscribers. Below is an overview of the core components involved in this process:

### Core Components

1. **ContentServer**: 
   - Acts as a central hub for managing communication between topics and subscribers.
   - Maintains a registry of subscribers for each topic.
   - Provides methods to publish messages to subscribers and manage subscriber registration.
  
   ```dart
   class ContentServer {
     // Singleton instance
     static final ContentServer _instance = ContentServer._();
     static ContentServer get instance => _instance;

     // Registers a subscriber to a topic
     void registerSubscriber(Subscriber sub) { ... }

     // Deregisters a subscriber from a topic
     void deRegisterSubscriber(Subscriber sub) { ... }

     // Publishes a message to all subscribers of a topic
     void publish(Topic topic, Packet message) { ... }
   }
   ```

2. **Subscriber**:
   - An abstract class that developers can extend to create custom subscribers.
   - Requires implementation of the `accept` method to handle incoming messages.
   - Each subscriber is associated with a specific topic and can be marked as permanent.

   ```dart
   abstract class Subscriber<T> implements Consumer<Packet> {
     // Topic associated with the subscriber
     Topic getTopic() { ... }

     // Method to handle incoming messages
     void accept(Packet packet);
   }
   ```

3. **Topic**
   - Set of predefined topics to categorize the types of data that can be published and subscribed to. Below is a list of available topics and their descriptions:

    ```dart
    enum Topic {
      // Raw ExG or impedance data.
      exg,

      // Orientation of the device.
      orientation,

      // Impedance measurement data.
      impedance,

      // Received command ack and status responses.
      command,

      // Information about the connected device.
      deviceInfo,

      // Environmental data.
      environment,

      // Processed and filtered ExG data.
      filteredExg,

      // Marker data.
      pushMarker,

      // Device calibration data - used in impedance calculations.
      calibrationInfo,
    }
    ```

## Disclaimer

The Mentalab Explore Pro API and hardware are intended strictly for research and educational applications.
