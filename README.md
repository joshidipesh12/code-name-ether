# code-name-ether

[![React Native](https://img.shields.io/badge/React_Native-20232A?style=flat&logo=react&logoColor=61DAFB)](#)
[![Expo](https://img.shields.io/badge/Expo-000020?style=flat&logo=expo&logoColor=white)](#)
[![Build Status](https://img.shields.io/badge/build-passing-brightgreen)](#)

**code-name-ether** is a privacy-first, zero-cloud security camera application designed to upcycle older Android and iOS devices. It transforms unused smartphones into headless, motion-activated security cameras that process and store video footage entirely on the local file system.

---

## 🚀 Key Features

*   **Zero-Cloud Privacy:** All footage is stored strictly on the local device. No servers, no accounts, no subscriptions.
*   **Intelligent Motion Detection:** Utilizes hyper-efficient OpenCV pixel-differencing to trigger recordings only when movement is detected, saving massive amounts of storage.
*   **Auto-Loop Recording (FIFO):** Seamlessly records in manageable video chunks. When the user-defined storage limit is reached, the app automatically deletes the oldest unprotected chunk.
*   **Extreme Power Saving Mode:** Designed for 24/7 headless operation. Triggers Immersive Mode, forces absolute minimum screen brightness, disables touch inputs, and turns off OLED pixels to prevent thermal throttling and battery degradation.
*   **Internal Gallery & Favorites:** Chronological grid of captured events. Users can lock up to 10 critical recordings to protect them from the auto-deletion loop.
*   **System Status Logs:** Real-time text output of device temperature, storage buffer, and foreground service health.

---

## 🏗️ Architecture & Custom Native Modules

This project uses **Expo Continuous Native Generation (CNG)** and strictly decouples its core engine into three highly optimized, cross-platform native modules. 

1.  `react-native-camera-foreground-service`
    *   Elevates app execution priority via Android's `FOREGROUND_SERVICE_TYPE_CAMERA`.
    *   Manages persistent system notifications to prevent aggressive OS battery optimizations from killing the recording loop.
2.  `react-native-directory-limit-manager`
    *   A high-performance C++/Kotlin file janitor.
    *   Bypasses the JS bridge to evaluate directory sizes and execute First-In, First-Out (FIFO) `.mp4` deletions securely.
3.  `react-native-opencv-motion-detector`
    *   A statically compiled C++ Nitro Module wrapping OpenCV (`cv::absdiff` and `cv::threshold`).
    *   Ingests YUV frames from `react-native-vision-camera`, dropping the color channel to minimize CPU overhead while identifying frame-by-frame pixel deviations.

---

## 🛠️ Getting Started

### Prerequisites
*   Node.js (v18+)
*   Android Studio / Xcode (for local native compilation)
*   Expo CLI

### Installation & Build

Because this project relies on custom native modules and C++ bindings, **standard Expo Go will not work.** You must compile the native Android/iOS application locally.

```bash
# 1. Clone the repository
git clone [https://github.com/your-org/code-name-ether.git](https://github.com/your-org/code-name-ether.git)
cd code-name-ether

# 2. Install dependencies
npm install

# 3. Prebuild the native directories (CNG)
npx expo prebuild --clean

# 4. Compile and run on a connected physical device (Emulators lack camera hardware)
npx expo run:android
# or
npx expo run:ios
```
---

## ⚙️ CI/CD & Testing

This repository utilizes GitHub Actions for Continuous Integration. Every Pull Request triggers a sandboxed environment that runs unit tests, lints the JavaScript/TypeScript layer, and executes a dry-run Gradle compilation. This automated pipeline ensures that any underlying C++ or Java/Kotlin modifications to the custom modules do not introduce complex Gradle build crashes into the main branch.

---

## 🗺️ Future Roadmap

•	Hybrid AI Pipeline: Implementing a cascaded logic flow where the lightweight OpenCV worklet triggers a localized TFLite MobileNet SSD model to verify human presence, drastically reducing false positives.

•	P2P WebRTC Access: Adding the ability to securely stream live footage to a secondary device over a direct peer-to-peer connection via self-hosted STUN/TURN servers.

## 🤝 Contributing

We welcome community contributions, especially for optimizing the C++ JSI bindings and iOS background audio workarounds for the foreground service. Please open an issue to discuss major architectural changes before submitting a Pull Request.

## 📜 License & Authorship

Distributed under the Apache 2.0 License. See LICENSE for more information.
