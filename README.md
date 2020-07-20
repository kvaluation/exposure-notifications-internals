# Exposure Notifications API

This repository contains a snapshot of code from Google Play Services' [Exposure Notifications
 module][1].

## Key Features

There are a number of features in this source set, including abstrations and JNI. The following sections provide key features with pointers to the source code.

### BLE MAC and RPI Rotation

Code: [`com.google.samples.exposurenotification.ble.advertising.BleAdvertiser#startAdvertising`][2]

Bluetooth Low Energy (BLE) MAC addresses rotate on average every 15 minutes to prevent remote location tracking that could be accomplished by tying together observations of a fixed MAC address.

In order to best protect user privacy, the Exposure Notifications framework ensures that Rolling
Proximity Identifiers (RPIs) are never rotated without also having a corresponding change of the
Bluetooth MAC address. For more details, see the full [Bluetooth spec][3].

Because Android doesn't have a callback to notify an application that the Bluetooth MAC address
is changing (or has changed), this is handled by explicitly stopping and restarting advertising
whenever a new RPI is generated.

Since there isn't any callback, it is possible for the Bluetooth MAC address to rotate
before a new RPI is generated. In this case the following would happen:

|Time | Bluetooth MAC | RPI |
|-----|---------------|-----|
| :00 | 00:00:00:01   | AAA |
| :09 | 00:00:00:02   | AAA |
| :10 | 00:00:00:03   | BBB |

The risk posed by this is minimal, since even though an observer may be able to tie the MAC
addresses `00:00:00:01` and `00:00:00:02` together using the common RPI, the duration of both
MAC addresses is no longer than a single RPI period (~10 minutes). When the RPI rotates, the
MAC address rotates again, making it difficult to track the association of either the MAC
address or RPI to a common device.

### Rolling Proximity Identifier Generation

Code: [`com.google.samples.exposurenotification.ble.advertising.RollingProximityIdManager#getCurrentRollingProximityId`](exposurenotification/src/main/java/com/google/samples/exposurenotification/ble/advertising/RollingProximityIdManager.java)
Code: [`com.google.samples.exposurenotification.data.generator.TemporaryExposureKeyGenerator`](exposurenotification/src/main/java/com/google/samples/exposurenotification/data/generator/TemporaryExposureKeyGenerator.java)

Rolling Proximity Identifiers (RPIs) are generated from a Temporary Exposure Key (TEK) based on the
[Exposure Notification Cryptography Specification][4].

For more information about TEKs and RPIs, see [Exposure Notifications Cryptography](CRYPTO.md#Temporary-Exposure-Key)

### Associated Encrypted Metadata

Code: [`com.google.samples.exposurenotification.ble.advertising.BleAdvertisementGenerator`](exposurenotification/src/main/java/com/google/samples/exposurenotification/ble/advertising/BleAdvertisementGenerator.java)
Code: [`com.google.samples.exposurenotification.data.generator.AssociatedEncryptedMetadataHelper`](exposurenotification/src/main/java/com/google/samples/exposurenotification/data/generator/AssociatedEncryptedMetadataHelper.java)

Associated Encrypted Metadata is generated by `BleAdvertisementGenerator#generatePacket` based on the
[Exposure Notification Cryptography Specification][4]. This 
uses `BluetoothMetadata#create` to create the metadata itself.

The encryption, decryption, and generation of keys is handled by the
class `AssociatedEncryptedMetadataHelper`.

For more information about Associated Encrypted Metadata, see [Exposure Notifications Cryptography](CRYPTO.md#Associated-Encrypted-Metadata)

### Key Matching

Code: [`com.google.samples.exposurenotification.matching.MatchingJni`](exposurenotification/src/main/java/com/google/samples/exposurenotification/matching/MatchingJni.java)

The core process of Key Matching is started via `MatchingJni`. The class then calls into native C++ code to improve performance by avoiding Java binder calls for the many crypto operations needed.

A `MatchingJni` object is initialized with a list of scanned RPIs from the past 14 days. The caller can pass a list of Diagnosis Key files, provided by the Healthcare Authority app, to the method `MatchingJni#matching`. This method produces a list of `TemporaryExposureKey`s corresponding to the Diagnosis Keys that the device was exposed to.

[1]: https://developers.google.com/android/exposure-notifications/exposure-notifications-api
[2]: exposurenotification/src/main/java/com/google/samples/exposurenotification/ble/advertising/BleAdvertiser.java
[3]: https://blog.google/documents/70/Exposure_Notification_-_Bluetooth_Specification_v1.2.2.pdf
[4]: https://blog.google/documents/69/Exposure_Notification_-_Cryptography_Specification_v1.2.1.pdf


