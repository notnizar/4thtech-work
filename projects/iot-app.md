# The 4Tech-IoT app and device firmware

[← Back to the overview](../README.md)

<img src="../assets/brand/4tech-iot-wordmark.png" alt="4Tech-IoT" width="320">

**Built:** August to October 2026 · **My part:** the app, the firmware, and the
[backend](iot-backend.md) they talk to

One Flutter app, for Android and iPhone, for every 4Tech smart-home device. It comes with
the ESP32 firmware the devices run.

## The main idea: the app knows no products

The app has no code for a water tank, a light or a plug. Each device's firmware publishes a
**schema**: a short description of its controls, its readings, its settings, and which
values need attention. The app builds the device's card, controls, settings and charts from
that.

A shortened schema for the water tank:

```json
{
  "type": "water_tank",
  "displayName": "Water Tank",
  "primary": "level",
  "traits": [
    {"key": "level", "type": "number", "label": "Fill Level", "unit": "%", "control": "gauge",
     "bands": [{"max": 10, "tone": "critical", "label": "Almost empty"},
               {"max": 25, "tone": "warning", "label": "Low"}]}
  ],
  "settings": [
    {"key": "tank_height_cm", "type": "number", "label": "Tank Height", "unit": "cm",
     "min": 10, "max": 2000, "default": 500}
  ]
}
```

From this the app shows a gauge in percent, colors it red below 10%, puts the tank on the
Home screen when it's low, and offers a Tank Height setting with the right limits. A new
product needs only new firmware. The app does not need an update, and nobody waits for a
store review.

## What the app does

- **Bluetooth setup.** The app finds a new device nearby, registers it to the account, and
  sends it the Wi-Fi and server settings. The Wi-Fi password is encrypted for that one
  device (ECDH P-256, then AES-256-GCM).
- **Control from anywhere.** Commands go through the server; state, command results and
  alerts come back live over a WebSocket that reconnects by itself.
- **Control without internet.** A device that loses the cloud for about 30 seconds starts
  advertising over Bluetooth again. The owner's app connects and controls it directly, with
  each command signed by a key only that device and its owner hold.
- **Sites and members.** An account belongs to one or more sites (a home, a farm, a shop).
  Owners add devices and invite people with one-time codes; members use the devices.
- **Alert rules.** "Tell me when the tank is below 20% for 10 minutes" or "when this device
  goes offline". The server checks the rules and sends a push notification, on Android and
  iPhone.
- **History.** Charts of each reading over the last 24 hours or 7 days.
- **Firmware updates.** The app shows the version a device runs, offers a newer one, and
  follows the installation.
- **Accounts.** Email and password, Sign in with Apple, Sign in with Google, password reset
  by emailed code, and account deletion from inside the app.
- **Demo mode.** From the sign-in screen, anyone can open the app with sample devices: no
  account, no hardware, nothing sent to the server. The demo runs the real screens against
  a stand-in backend inside the app.

## How the app is built

- **Flutter**, with Riverpod for state. The code is organized by feature: alerts, auth,
  dashboard, devices, onboarding, provisioning, settings, sites.
- **28 test files** cover the Bluetooth framing, the setup cryptography, the schema parser,
  device sync, alerts, sign-in and more. CI runs the analyzer and the tests before every
  release build.
- **Screens rendered to images in tests.** A test tool draws about fifty screen states
  (light and dark, iOS and Android, text at 200%) to PNG files, so the design can be
  reviewed without a phone.
- **Release builds from CI** (Codemagic): a signed bundle for Google Play and a signed build
  uploaded to App Store Connect, with the Dart code obfuscated and the symbols kept so
  crash reports can be read back.
- **Release builds refuse a plain-HTTP server**, and trust only the phone's own root
  certificates.

## The firmware

Every product is a small sketch on top of a shared library I wrote, **SmartHomeCore**. The
library does what all devices share:

| Part | What it does |
|---|---|
| Identity | On first start the device creates its own private key. The key never leaves the device; the server certifies it from a signing request. |
| Setup | Bluetooth setup, with the Wi-Fi password and the server token encrypted in transit |
| Cloud | Wi-Fi that keeps retrying after power cuts; MQTT over mutual TLS; a "last will" message so the server knows when the device drops |
| Offline control | Signed commands over Bluetooth when the cloud is unreachable |
| Settings | Declared in the schema; the library stores them, checks new values against their limits and reports them |
| Updates | Over-the-air updates. The device checks the file's hash and signature, installs it beside the running version, and goes back to the old version if the new one can't reach the cloud within 10 minutes |

With the library, a whole new product is about twenty lines: a schema, a handler for
commands, and a call to report new readings.

Products so far:

| Firmware | Hardware |
|---|---|
| Water tank, ultrasonic | ESP32 with an HC-SR04 distance sensor |
| Water tank, two units | A tank unit with a depth probe and an NRF24 radio, and an indoor gateway with an NRF24 receiver, an OLED screen, Wi-Fi and Bluetooth |
| Smart light | A bare ESP32, its on-board LED as a dimmable light, for testing without sensors |

The device cryptography (key, signing request, certificate check, setup encryption, command
signature) has an interoperability test that runs the firmware's code against the server's
code.

## Where it started

The first version, in August 2026, talked to a public MQTT broker straight from the phone
and used an unauthenticated Bluetooth protocol. Version 2 replaced all of that: the
self-hosted [backend](iot-backend.md), accounts, mutual TLS, encrypted setup and signed
commands. The problems found along the way are in [problems-solved.md](../problems-solved.md).
