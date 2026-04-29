# Explore Core SDK (Add-to-App)

`Explore Core SDK` enables integration of Explore device functionality into mobile products built with Flutter or native mobile architectures.

## Functionalities

- Bluetooth device discovery (scan for supported devices)
- Device connection and disconnection management
- Real-time ExG, orientation and marker data streaming
- Real-time impedance measurement
- Local session recording
- LSL support

## Integration

For add-to-app architectures, a common setup is:

1. Embed a Flutter runtime/module inside your native mobile app
2. Expose a bridge layer between native and Explore core dart SDK operations
3. Trigger SDK actions from your product flows (scan, connect, stream, record)
4. Surface runtime status/events in your app UI and logs
5. Add production-grade permission handling, lifecycle management, and error recovery

If you need implementation support, integration guidance, or access details, please contact us.
