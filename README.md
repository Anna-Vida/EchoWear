# EchoWear

<p align="center">
  <img src="assets/echowear.png" alt="EchoWear logo" width="340" />
</p>

<p align="center">
  <strong>Smart wearable glove and mobile application for two-way Filipino Sign Language communication.</strong>
</p>

<p align="center">
  React Native · Expo · Bluetooth Low Energy · ESP32 · Flex Sensors · MPU-6050 · LSTM · TensorFlow Lite · Speech
</p>

---

## Overview

**EchoWear** is a collaborative capstone project that combines wearable hardware, sensor data, machine learning, and a mobile application to support two-way communication involving Filipino Sign Language (FSL).

The system is designed around two communication directions:

1. **FSL → Speech** — the glove captures finger-bend and hand-motion data, the mobile application receives the sensor stream through Bluetooth Low Energy (BLE), an on-device TensorFlow Lite model classifies the gesture, and the detected result can be spoken aloud.
2. **Speech → Text** — the mobile application also provides speech-recognition functionality so spoken input can be converted into text for the user.

<p align="center">
  <img src="echowear prototype.png" alt="EchoWear wearable glove prototype" width="320" />
</p>

---

## What I worked on

This repository represents a **team project with multiple contributors**. The Git history and GitHub contributor list should be used for the complete collaboration record.

### Anna Patricia Vida — hardware, data, ML integration, and app connectivity

My work on EchoWear included:

- Building and wiring the wearable glove hardware.
- Integrating the flex sensors and motion sensor with the ESP32-based controller.
- Collecting sensor readings used for gesture data.
- Preparing the sensor-data flow used by the application and model.
- Working on the **LSTM training workflow** using the collected sequential sensor data.
- Connecting the glove to the React Native application through **Bluetooth Low Energy**.
- Implementing the data path from the physical glove into the application.
- Matching application-side preprocessing with the preprocessing used during model training.
- Integrating the trained model into the mobile workflow for real-time gesture prediction.
- Connecting the prediction result to the application's FSL-to-speech experience.

> EchoWear was developed collaboratively. This section documents my own contributions and is not intended to replace or minimize the work of the other contributors.

[View the complete GitHub contributor history](https://github.com/Anna-Vida/EchoWear/graphs/contributors)

---

## System Architecture

```text
┌──────────────────────── EchoWear Glove ────────────────────────┐
│                                                                 │
│  5 Flex Sensors ─┐                                              │
│                  ├──> ESP32-based controller ───> BLE Service   │
│  MPU-6050 ───────┘         sensor sampling                      │
│   motion data                                                    │
└───────────────────────────────────────┬─────────────────────────┘
                                        │
                                        │ Bluetooth Low Energy
                                        ▼
┌──────────────────────── Mobile Application ─────────────────────┐
│                                                                 │
│  BLE scan → connect → receive sensor rows                        │
│                         │                                        │
│                         ▼                                        │
│                 Validate 11 features                             │
│              5 flex + 6 motion values                            │
│                         │                                        │
│                         ▼                                        │
│                    Normalize data                                │
│                         │                                        │
│                         ▼                                        │
│              60-frame sliding window                             │
│                         │                                        │
│                         ▼                                        │
│               Flatten to 660 values                              │
│                         │                                        │
│                         ▼                                        │
│                  TensorFlow Lite                                 │
│               gesture classification                            │
│                         │                                        │
│                         ▼                                        │
│                    FSL result                                    │
│                         │                                        │
│                         ▼                                        │
│                   Text-to-Speech                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Hardware

The wearable prototype combines finger-bend sensing and hand-motion sensing so a gesture is represented by both hand shape and movement.

### Main hardware

| Component | Purpose |
| --- | --- |
| ESP32-based controller | Reads sensors and sends the data wirelessly |
| 5 flex sensors | Measures the bend of the thumb, index, middle, ring, and pinky |
| MPU-6050 | Captures hand motion/orientation data |
| Glove | Wearable base for the sensing system |
| Wiring / resistors / prototyping board | Sensor connections and signal conditioning |

The five flex sensors provide finger-bend information while the MPU-6050 provides motion information. Together, those values create the feature vector used by the recognition pipeline.

---

## Sensor Data Format

The application currently expects **11 values per sensor frame**:

```text
5 flex-sensor values + 6 MPU values = 11 features
```

The BLE hook validates incoming rows before they are accepted. Invalid or incomplete rows are ignored rather than being sent to the model.

The application keeps a sliding sequence of:

```text
60 timesteps × 11 features = 660 input values
```

This sequence-based input is important because the recognition problem is not only about a single hand position. The model can use changes across time to recognize motion patterns.

---

## BLE Connection: Glove → Mobile App

The React Native application uses `react-native-ble-plx`.

Current BLE identifiers in the application:

```text
Service UUID:
4fafc201-1fb5-459e-8fcc-c5c9c331914b

Characteristic UUID:
beb5483e-36e1-4688-b7f5-ea07361b26a8
```

### Connection flow

1. The app requests the required Android Bluetooth permissions.
2. It scans specifically for devices advertising the EchoWear BLE service.
3. The selected device is connected.
4. Services and characteristics are discovered.
5. The app requests a larger MTU for the connection.
6. The app monitors the EchoWear characteristic.
7. Incoming Base64 BLE data is decoded into UTF-8 text.
8. Comma-separated values are parsed into numeric sensor readings.
9. The row is validated, normalized, and added to the sliding sequence buffer.
10. Once the buffer reaches 60 frames, the sequence is prepared for TensorFlow Lite inference.

Implementation: [`src/components/useBluetooth.js`](src/components/useBluetooth.js)

---

## Data Preprocessing

One of the important integration details is that **mobile preprocessing must match the preprocessing used during model training**.

The current app normalizes incoming sensor data as follows:

### Flex sensors

```text
normalized = raw value / 100
```

Values are clamped to the `0..1` range.

### MPU values

```text
normalized = (value + 1) / 2
```

These values are also clamped to `0..1`.

After normalization, each row is stored in the sliding buffer. When 60 frames are available, the app builds a `Float32Array` containing 660 values for inference.

---

## LSTM Training Workflow

The recognition model was trained using the sequential sensor data collected from the EchoWear glove.

Conceptually, the workflow is:

```text
Wear FSL glove
      ↓
Perform labeled gesture
      ↓
Collect flex + MPU sensor values
      ↓
Save labeled time-series samples
      ↓
Clean / normalize data
      ↓
Create fixed-length sequences
      ↓
Train LSTM model
      ↓
Evaluate and refine
      ↓
Export model for mobile inference
      ↓
TensorFlow Lite model in EchoWear app
```

### Why LSTM?

EchoWear works with **time-series sensor data**. A gesture can contain movement over multiple frames, so sequence order matters. An LSTM network is useful for learning patterns across a sequence rather than evaluating each reading independently.

The deployed application contains the TensorFlow Lite model at:

[`assets/algorithm/echowear_model.tflite`](assets/algorithm/echowear_model.tflite)

and its ordered class labels at:

[`assets/algorithm/labels.json`](assets/algorithm/labels.json)

> The model-training notebook/script is not currently documented in this repository. If the final training notebook is added later, this section should link directly to it so the complete training process is reproducible.

---

## On-Device Inference

The mobile application uses `react-native-fast-tflite` to load and execute the EchoWear model locally.

The inference flow is:

```text
BLE sensor sequence
      ↓
normalized Float32 input
      ↓
TensorFlow Lite model
      ↓
class confidence scores
      ↓
highest-confidence class
      ↓
confidence threshold
      ↓
recognized FSL output
```

The current implementation uses a confidence threshold of **0.40** before showing a gesture as a prediction.

Implementation: [`src/components/Model.js`](src/components/Model.js)

---

## FSL → Speech

When the model produces a recognized label:

1. The predicted gesture is displayed in the **FSL to Speech** card.
2. The user can trigger speech output.
3. The app uses its text-to-speech component to vocalize the recognized result.

This creates the main path:

```text
FSL gesture → glove sensors → BLE → LSTM/TFLite → text → speech
```

---

## Speech → Text

EchoWear also includes the opposite communication direction in the mobile interface.

The Home screen integrates a speech-recognition flow that allows speech to be transcribed into text. A manual text-entry mode is also available in the same interface.

This creates the complementary path:

```text
spoken input → speech recognition → displayed text
```

---

## Mobile Application

The application is built with:

- React Native
- Expo
- React Navigation
- `react-native-ble-plx`
- `react-native-fast-tflite`
- Expo Speech
- React Native / Expo audio and file-system utilities

Key files:

| File | Responsibility |
| --- | --- |
| [`src/components/useBluetooth.js`](src/components/useBluetooth.js) | BLE scanning, connection, sensor parsing, normalization, buffering |
| [`src/components/Model.js`](src/components/Model.js) | Loads the TFLite model and performs inference |
| [`src/screens/Home.js`](src/screens/Home.js) | Main connection, FSL-to-speech, and speech-to-text UI |
| [`assets/algorithm/echowear_model.tflite`](assets/algorithm/echowear_model.tflite) | Deployed gesture-recognition model |
| [`assets/algorithm/labels.json`](assets/algorithm/labels.json) | Ordered model output labels |

---

## Prototype & Application Screens

### Wearable prototype

<p align="center">
  <img src="echowear prototype.png" alt="EchoWear wearable glove prototype" width="320" />
</p>

The physical prototype integrates the glove, flex sensors, motion sensing, controller, wiring, and the BLE link used to send live sensor readings to the mobile application.

### Mobile application

<p align="center">
  <img src="echowear 3.jpg" alt="EchoWear demo mode" width="260" />
  &nbsp;&nbsp;
  <img src="echowear 2.jpg" alt="EchoWear connected glove state" width="260" />
  &nbsp;&nbsp;
  <img src="echowear 1.jpg" alt="EchoWear FSL alphabet recognition" width="260" />
</p>

The screenshots show the application in demo mode, a live connected-glove state, and FSL recognition output. The interface combines **FSL to Speech** with **Speech to Text** so the glove and mobile app can support two-way communication.

---

## Research Paper

The final EchoWear research/capstone paper should be stored in:

```text
docs/EchoWear-Research-Paper.pdf
```

A direct link will be added here once the PDF is committed to the repository.

**Status:** paper file still needs to be uploaded.

---

## Running the Mobile App

### Requirements

- Node.js
- npm
- Expo-compatible development environment
- Android device for the BLE workflow
- EchoWear glove for live sensor input

### Install

```bash
npm install
```

### Start Expo

```bash
npm start
```

or:

```bash
npx expo start
```

Because the project uses native BLE and TensorFlow Lite libraries, a development build may be required instead of Expo Go for all hardware/model functionality.

---

## Repository Structure

```text
EchoWear/
├── App.js
├── assets/
│   ├── algorithm/
│   │   ├── echowear_model.tflite
│   │   └── labels.json
│   ├── echowear.png
│   └── glove.png
├── src/
│   ├── components/
│   │   ├── Model.js
│   │   ├── useBluetooth.js
│   │   ├── useSpeechToText.js
│   │   └── useTextToSpeech.js
│   └── screens/
│       └── Home.js
├── package.json
└── README.md
```

---

## Contributors

EchoWear is a collaborative project with contributions across hardware, mobile development, machine learning, research, testing, and integration.

Rather than manually claiming ownership of every area, this README separates documented personal contributions from the repository's complete Git history.

**Complete contributor record:**  
[github.com/Anna-Vida/EchoWear/graphs/contributors](https://github.com/Anna-Vida/EchoWear/graphs/contributors)

---

## Project Status

EchoWear is an academic/research prototype. Hardware behavior, sensor calibration, model accuracy, BLE reliability, and recognition performance can vary between devices, users, environments, and glove builds.

---

## License / Academic Use

No explicit open-source license is currently included in this repository. Unless a license is added by the project owners, the source code and project materials should not be assumed to grant unrestricted reuse, redistribution, or modification.
