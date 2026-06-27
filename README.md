# Code Name: Ether 🛡️📱

**An Open-Source, Zero-Cloud Local DVR Security Camera App**

Code Name: Ether is a privacy-first mobile application designed to upcycle your old, unused Android and iOS smartphones into reliable, headless security cameras. Unlike traditional cloud-dependent security solutions, Ether processes and stores all video footage locally on the device. 

Built with a focus on long-term hardware sustainability and zero-cloud architecture, it is the perfect solution for tech-savvy and privacy-conscious users who want complete control over their security data.

## 🌟 Key Features (MVP)

*   **100% Local Storage:** Videos are recorded directly to the device's storage in segmented chunks without any cloud uploading. 
*   **Motion-Activated Recording:** Saves battery and storage by only recording when pixel-level motion is detected using highly-optimized OpenCV algorithms.
*   **Intelligent Loop Recording (FIFO):** Acts as an automated janitor, actively monitoring the allocated storage limit and automatically deleting the oldest video chunks to maintain a continuous recording loop.
*   **Extreme Power-Saving Mode:** Designed to run 24/7 on older hardware without causing thermal throttling or screen burn-in. The app utilizes Android's Immersive Mode, forces the lowest brightness, disables UI inputs, and turns off OLED pixels to keep the phone running cool.
*   **Internal App Gallery:** An organized chronological grid of motion events allowing users to view, export, and delete recordings. 
*   **Protected Favorites:** Users can "lock" up to 10 important video recordings to prevent them from being overwritten by the auto-deletion loop.
*   **Live System Status & Logs:** A dedicated dashboard detailing real-time device temperature, remaining storage buffers, and operational logs (e.g., motion detected, background services healthy) to guarantee headless reliability.
*   **Advanced Camera Setup Overlay:** Granular controls for stream resolution (480p/1080p), flashlight toggles, orientation lock, and active motion indicators.

## 🏗️ Technical Architecture 

To achieve system-level hardware control while maintaining a fluid user interface, Ether is built using **Expo** with **Continuous Native Generation (CNG)**. 

### Core Tech Stack
*   **UI & Routing:** React Native, Expo Router
*   **Camera Hardware Interface:** `react-native-vision-camera`
*   **File Operations:** `expo-file-system`

### Custom Native Modules
Because standard background libraries cannot handle continuous hardware access, Ether heavily relies on three custom, deeply decoupled Expo/React Native modules:

1.  **`react-native-camera-foreground-service`**: Prevents the Android OS from killing the background camera process. It wraps the standard Android Foreground Service with the `FOREGROUND_SERVICE_TYPE_CAMERA` declaration and manages persistent notifications to ensure the app stays alive indefinitely.
2.  **`react-native-directory-limit-manager`**: A high-performance native directory watcher that bypasses the JS bridge to query the file system. It enforces the maximum storage limits using a First-In, First-Out (FIFO) deletion script while ignoring user-favorited files.
3.  **`react-native-opencv-motion-detector`**: A highly optimized C++ Frame Processor built via Nitro Modules. By analyzing the lightweight YUV luminance channel utilizing OpenCV (`cv::absdiff` and `cv::threshold`), it executes hyper-efficient pixel differencing for continuous motion detection without draining the battery.

## 🚀 Future Roadmap

While the MVP focuses on a purely local, zero-cloud architecture, the underlying infrastructure is designed to accommodate powerful future expansions:

*   **Hybrid AI Motion Pipeline:** Implementing a lightweight TFLite model (e.g., MobileNet SSD) to validate human presence. The C++ OpenCV pixel differencing will serve as a low-power "wake-up" trigger for the AI, drastically reducing false positives while maintaining battery health.
*   **P2P Live Streaming:** Integration of WebRTC with lightweight STUN/TURN signaling servers to allow remote, end-to-end encrypted live viewing without centralized cloud servers.

## 🤝 Contributing
Code Name: Ether is an open-source project built for the community. Since we leverage custom Expo Native Modules and heavy C++ bindings, please refer to our contributing guidelines before opening PRs regarding Gradle or CMake modifications. 

## 📝 License
Distributed under the Apache 2.0 License. See LICENSE for more information.
