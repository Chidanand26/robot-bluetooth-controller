# Robot Bluetooth Controller — Android App

An Android application (Kotlin) for driving and monitoring a robot over a Bluetooth
serial (SPP) link, with manual D-pad control, autonomous navigation commands, and a
live sensor dashboard.

[![Kotlin](https://img.shields.io/badge/Kotlin-100%25-7F52FF?logo=kotlin&logoColor=white)](https://kotlinlang.org/)
[![Android](https://img.shields.io/badge/Android-minSdk%2021-3DDC84?logo=android&logoColor=white)](https://developer.android.com/)
[![Gradle](https://img.shields.io/badge/Build-Gradle%20Kotlin%20DSL-02303A?logo=gradle&logoColor=white)](https://gradle.org/)
[![Bluetooth](https://img.shields.io/badge/Connectivity-Bluetooth%20SPP-0082FC?logo=bluetooth&logoColor=white)](https://developer.android.com/develop/connectivity/bluetooth)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

---

## Overview

The app connects to a robot or microcontroller exposing a Bluetooth **Serial Port
Profile (SPP)** service and sends newline-terminated ASCII command strings (for
example `FORWARD:128`, `STOP`, `START_AUTO`). A bottom-navigation UI switches between
three screens: **Manual**, **Auto**, and **Sensors**. A single shared `RobotViewModel`
holds connection state and dispatches commands, so all screens act on the same
connection.

---

## Features

- **Bluetooth device pairing & connection** — lists the phone's already-paired devices
  in a picker dialog and connects to the chosen one over the standard SPP UUID
  (`00001101-0000-1000-8000-00805F9B34FB`). Connect/Disconnect is toggled from a
  status bar that reflects live connection state.
- **Runtime permission handling** — requests `BLUETOOTH_CONNECT` / `BLUETOOTH_SCAN`
  on Android 12+ and legacy `BLUETOOTH` / `BLUETOOTH_ADMIN` + `ACCESS_FINE_LOCATION`
  on older versions.
- **Manual drive control** — forward / backward / left / right D-pad buttons that
  send a movement command on press and a `STOP` on release, an adjustable speed
  slider (0–255), plus dedicated **Stop**, **Emergency Stop**, and **Reset** buttons.
- **Autonomous mode** — set an (X, Y) target (manually or via Home / Point-1 /
  Point-2 presets), start/stop autonomous navigation, and start/stop a built-in
  **Figure-8** pattern. The screen also displays position, orientation, mode, and
  battery readouts from the robot state.
- **Sensor dashboard** — displays left/right ultrasonic distances, eight IR sensor
  values, position, orientation, battery voltage, current mode, and a rolling
  command history; includes a "Request Data" button that asks the robot for a
  status update.
- **Bottom-navigation UI** — Material bottom navigation between Manual, Auto, and
  Sensors fragments.

> **Honesty note:** The app reliably **sends** commands over Bluetooth. The inbound
> read loop in `BluetoothService` receives bytes from the robot, but the current code
> does not yet parse that stream into `RobotState`. As a result, the sensor / position
> / battery fields render the `RobotViewModel` default values until an inbound parser
> is implemented (see *Future Improvements*). The values shown for those fields are
> therefore placeholders, not yet live telemetry.

---

## Tech Stack

| Area              | Technology                                              |
|-------------------|---------------------------------------------------------|
| Language          | Kotlin                                                  |
| Platform          | Android (minSdk 21, targetSdk / compileSdk 34)          |
| UI                | Android Views + ViewBinding, Material Components, Fragments, Bottom Navigation |
| Architecture      | Activity + Fragments + `ViewModel` + `LiveData`         |
| Connectivity      | Android Bluetooth classic, RFCOMM socket over SPP UUID  |
| Concurrency       | Kotlin Coroutines (`Dispatchers.IO`) for connect/read   |
| Build             | Gradle (Kotlin DSL), version catalog (`libs.versions.toml`) |

---

## App Architecture

```
                         +------------------------+
                         |     MainActivity       |
                         |  - bottom navigation   |
                         |  - connect button      |
                         |  - permission requests |
                         +-----------+------------+
                                     |
                hosts fragments      |     owns / observes
              +----------------------+----------------------+
              |                      |                      |
      +-------v------+      +--------v-------+      +-------v-------+
      | ManualFrag.  |      |   AutoFrag.    |      |  SensorFrag.  |
      | D-pad, speed |      | target, fig-8  |      | sensor readout|
      +-------+------+      +--------+-------+      +-------+-------+
              |                      |                      |
              |   all call shared    |                      |
              +----------+-----------+----------------------+
                         |
                  +------v---------------------+
                  |       RobotViewModel       |
                  | - connectionStatus (Live)  |
                  | - robotState (LiveData)    |
                  | - commandHistory (LiveData)|
                  | - moveForward()/stop()/... |
                  +------------+---------------+
                               | sendCommand("FORWARD:128")
                  +------------v---------------+
                  |      BluetoothService      |
                  | - RFCOMM socket (SPP UUID) |
                  | - coroutine connect + read |
                  | - sendCommand() => bytes   |
                  +------------+---------------+
                               | Bluetooth serial (SPP / RFCOMM)
                               | ASCII line:  "<COMMAND>\n"
                  +------------v---------------+
                  |  Robot / microcontroller   |
                  |  (HC-05 / HC-06 style BT    |
                  |   serial module + firmware) |
                  +----------------------------+
```

**Command protocol (sent by the app):** each command is an ASCII string terminated by
`\n`. Examples from `RobotViewModel`:

| Command string        | Meaning                          |
|-----------------------|----------------------------------|
| `FORWARD:<speed>`     | Drive forward at speed 0–255     |
| `BACKWARD:<speed>`    | Drive backward                   |
| `LEFT:<speed>` / `RIGHT:<speed>` | Turn                  |
| `STOP`                | Stop motion                      |
| `EMERGENCY_STOP`      | Immediate stop                   |
| `TARGET:<x>,<y>`      | Set navigation target            |
| `START_AUTO` / `STOP_AUTO` | Autonomous navigation       |
| `START_FIGURE8` / `STOP_FIGURE8` | Figure-8 pattern      |
| `REQUEST_STATUS`      | Ask robot for a status update    |
| `RESET`               | Reset robot                      |

The robot firmware is expected to interpret these strings; the firmware itself is not
part of this repository.

---

## Project Structure

```
Bot_Controller/
├── app/
│   └── src/main/
│       ├── AndroidManifest.xml
│       ├── java/com/vibeshift/robotcontroller/
│       │   ├── MainActivity.kt          # host activity, nav, BT setup, permissions
│       │   ├── BluetoothService.kt      # RFCOMM connect/read/send over SPP
│       │   ├── RobotViewModel.kt        # shared state + command API
│       │   ├── DeviceListDialog.kt      # paired-device picker dialog
│       │   ├── bottom_nav.kt            # secondary bottom-nav activity
│       │   ├── fragments/
│       │   │   ├── ManualFragment.kt    # D-pad + speed control
│       │   │   ├── AutoFragment.kt      # target / autonomous / figure-8
│       │   │   └── SensorFragment.kt    # sensor + telemetry dashboard
│       │   └── ui/
│       │       ├── theme/               # Compose theme (Color/Theme/Type)
│       │       ├── home/                # generated nav template screens
│       │       ├── dashboard/
│       │       └── notifications/
│       └── res/
│           ├── layout/                  # activity & fragment layouts
│           ├── menu/bottom_nav_menu.xml
│           ├── navigation/
│           ├── drawable/  drawable-v24/
│           ├── mipmap-*/                # launcher icons (.webp)
│           └── values/                  # strings, colors, themes, dimens
├── build.gradle.kts
├── settings.gradle.kts
├── gradle/  (wrapper + libs.versions.toml)
├── gradlew  gradlew.bat
├── LICENSE
└── README.md
```

> Note: the project mixes the classic Views/ViewBinding UI (the active app screens)
> with a Compose theme and the Android "Bottom Navigation Activity" template
> (`ui/home`, `ui/dashboard`, `ui/notifications`, `bottom_nav.kt`). The Views-based
> Manual/Auto/Sensor flow is the functional controller; the template screens remain
> from project scaffolding.

---

## Hardware Setup

This app targets a robot that exposes a **Bluetooth classic serial port (SPP /
RFCOMM)** — the connection uses the well-known SPP UUID
`00001101-0000-1000-8000-00805F9B34FB`, which is the profile used by common hobby
Bluetooth-serial modules such as **HC-05 / HC-06** paired with a microcontroller
(e.g. Arduino) driving the motors and sensors.

Typical setup the app is written against:

- A Bluetooth serial module (HC-05 / HC-06 or equivalent SPP device) wired to a
  microcontroller's UART.
- Microcontroller firmware that reads the newline-terminated command strings above
  and drives motors accordingly (and, ideally, streams telemetry back).
- Sensors referenced by the UI: two ultrasonic distance sensors (left/right) and an
  array of eight IR sensors, plus battery-voltage reporting.

> The robot firmware and wiring are **not** included in this repository; the exact
> sensor/motor hardware is inferred from the command set and sensor fields in the
> Android code. Pair the module with the phone in Android Bluetooth settings before
> launching the app — the app connects to already-paired devices.

---

## Installation / Build

**Requirements**

- Android Studio (Giraffe or newer recommended)
- JDK 17 (the project compiles with `sourceCompatibility`/`targetCompatibility` 17)
- Android SDK with API 34
- An Android device running **Android 5.0 (API 21) or higher** with Bluetooth

**Build from Android Studio**

1. Clone the repository and open the `Bot_Controller` folder in Android Studio.
2. Let Gradle sync (the Gradle wrapper handles the Gradle version).
3. Connect an Android device (a physical device is recommended — Bluetooth is not
   available on most emulators) and click **Run**.

**Build from the command line**

```bash
# from the Bot_Controller directory
./gradlew assembleDebug        # builds app/build/outputs/apk/debug/app-debug.apk
./gradlew installDebug         # builds and installs on a connected device
```

(Use `gradlew.bat` on Windows.)

---

## Usage

1. In your phone's system Bluetooth settings, **pair** the robot's Bluetooth module.
2. Launch the app and grant the requested Bluetooth/location permissions.
3. Tap **Connect** and pick the robot from the paired-device list; the status bar
   shows **Connected** (green) when the link is open.
4. **Manual** tab: hold the directional buttons to drive (releasing sends `STOP`),
   adjust the speed slider, and use Stop / Emergency Stop / Reset as needed.
5. **Auto** tab: enter an X/Y target (or use a preset), start/stop autonomous
   navigation, or run the Figure-8 pattern.
6. **Sensors** tab: tap **Request Data** to ask the robot for a status update and
   view sensor / telemetry fields and recent command history.

---

## Screenshots

Screenshots are not committed yet. Add captures of the Manual, Auto, and Sensors
screens to `docs/screenshots/` and reference them here, for example:

```
https://placehold.co/640x360/1f2937/a3e635?text=Manual+Mode
https://placehold.co/640x360/1f2937/a3e635?text=Auto+Mode
https://placehold.co/640x360/1f2937/a3e635?text=Sensor+Dashboard
```

<!--
![Manual control](https://placehold.co/640x360/1f2937/a3e635?text=Manual+Mode)
![Auto mode](https://placehold.co/640x360/1f2937/a3e635?text=Auto+Mode)
![Sensors](https://placehold.co/640x360/1f2937/a3e635?text=Sensor+Dashboard)
-->

---

## Future Improvements

- **Parse inbound telemetry** — extend `BluetoothService.startReading()` to decode the
  incoming serial stream and update `RobotState` so the Sensors/Auto screens show live
  ultrasonic, IR, position, orientation, and battery data instead of defaults.
- **Device discovery** — add active scanning/pairing of new devices in-app (currently
  only already-paired devices are listed).
- **Connection resilience** — auto-reconnect and clearer error feedback on dropped
  links.
- **Consolidate the UI** — remove the unused bottom-nav template screens
  (`ui/home`, `ui/dashboard`, `ui/notifications`) or migrate the whole app to a single
  UI toolkit.
- **Hook up "Clear History"** on the Sensors screen (currently a no-op placeholder).
- **Configurable command protocol** so the app can adapt to different robot firmwares.

---

## License

This project is licensed under the MIT License — see the [LICENSE](LICENSE) file for
details.

---

## Author

**Chidanand S Karennavar**
GitHub: [@Chidanand26](https://github.com/Chidanand26)

---

## Contributing

Contributions are welcome. Please open an issue to discuss a change, then submit a
pull request:

1. Fork the repository.
2. Create a feature branch (`git checkout -b feature/my-change`).
3. Commit your changes.
4. Open a pull request describing what and why.
