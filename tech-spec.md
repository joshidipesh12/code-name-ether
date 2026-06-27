# Code Name: Ether 🛡️📱

**An Open-Source, Zero-Cloud Local DVR Security Camera App**

Code Name: Ether is a privacy-first mobile application designed to upcycle your old, unused Android and iOS smartphones into reliable, headless security cameras. Unlike traditional cloud-dependent security solutions, Ether processes and stores all video footage locally on the device. 

Built with a focus on long-term hardware sustainability and completely open-source transparency, it is the perfect solution for tech-savvy and privacy-conscious users who want complete control over their security data and application binaries.

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
*(Add appropriate open-source license information here)*

---

# Technical Approach and Implementation Details

## 1. Architectural Strategy

The application relies on deep hardware integration (camera, file system, background services) which exceeds the capabilities of standard React Native libraries. To achieve this, the architecture is built on the **Expo** framework using **Continuous Native Generation (CNG)** and local **Development Builds**.

*   **Development Builds over Expo Go:** Because standard Expo Go does not support custom native code for heavy background processing and C++ bindings, the app utilizes `expo-dev-client` to create a custom development build.
*   **Continuous Native Generation (CNG):** Native Android and iOS directories are generated on-demand from the `app.json` configuration and `package.json`. This allows the team to maintain a clean JavaScript/TypeScript codebase while safely managing native dependencies through Config Plugins.
*   **Expo Modules API:** Custom native features (OpenCV, background processes) are written using the Expo Modules API, which offers a modern, idiomatic Swift and Kotlin DSL that is heavily optimized through React Native's JSI (JavaScript Interface).

## 2. Phase 1: Project Initialization & Environment Setup

The development lifecycle begins by scaffolding a universally compatible React Native application using Expo CLI.

1.  **Initialize the Project:**
    Bootstrap the project utilizing the default Expo SDK template with built-in TypeScript support.
    ```bash
    npx create-expo-app@latest --template default@sdk-56
    ```
2.  **Install Development Client:**
    Install `expo-dev-client` to allow local native compilation and access to the developer menu.
    ```bash
    npx expo install expo-dev-client
    ```
3.  **Local App Compilation:**
    Generate the native code locally to test hardware-specific features without relying on cloud services.
    ```bash
    npx expo run:android
    npx expo run:ios
    ```

## 3. Phase 2: Navigation & UI Layer

**Technology:** Expo Router

Ether uses **Expo Router** for file-based routing, which organizes navigation strictly through the file system structure within the `src/app` directory.

*   **Structure:**
    *   `src/app/_layout.tsx`: The root layout for shared UI elements and context providers.
    *   `src/app/index.tsx`: The Home dashboard featuring live camera feed and system status.
    *   `src/app/gallery/index.tsx`: The internal chronological video gallery.
    *   `src/app/settings.tsx`: Settings page for configurations.

## 4. Phase 3: Custom Native Modules (Core Engine)

The core DVR engine will be broken out into three highly decoupled Expo Native Modules. These are initialized within the project using the local module generator:
```bash
npx create-expo-module@latest --local
```

*   **Module 1: `react-native-camera-foreground-service`**: Wraps the standard Android Foreground Service with the `FOREGROUND_SERVICE_TYPE_CAMERA` declaration.
*   **Module 2: `react-native-directory-limit-manager`**: A native-level directory watcher enforcing storage limits via a First-In, First-Out (FIFO) deletion script bypassing the JS bridge entirely.
*   **Module 3: `react-native-opencv-motion-detector`**: Developed via a C++ JSI Worklet integrating **OpenCV**. It evaluates the YUV luminance channel frame-by-frame from `react-native-vision-camera`.

## 5. Phase 4: File System & Storage Integration

**Technology:** `expo-file-system`, SQLite

Ether records video in smaller, discrete chunks (e.g., `motion_event_144507.mp4`).
*   **Video Saving:** Completed chunks are written directly to the device's local application storage using `expo-file-system`.
*   **Metadata Storage:** A local database of metadata for the segmented chunks is maintained utilizing `expo-sqlite`.
*   **Persistent Configuration:** Small variables like user settings are stored locally using `expo-secure-store`.

## 6. Phase 5: UI Polish & Immersive Power-Saving Mode

A core feature of Ether is the extreme power-saving mode to prevent screen burn-in on older devices. This requires deep control over the operating system's UI overlays.

*   **Status Bar Control:** We use the `expo-status-bar` library to hide the top system icons while the app is actively recording.
*   **Safe Area Handling:** When users access the Settings or Gallery, we use the `SafeAreaView` from `react-native-safe-area-context` to ensure UI controls render safely within the device's physical notches.

## 7. Phase 6: Local Production Builds (Strictly Open-Source)

To remain strictly independent of cloud providers, all compilation, code signing, and artifact generation are handled on local hardware.

### Android Release Build (APK / AAB)
1.  **Generate Native Code:** Run `npx expo prebuild` to create the `/android` directory.
2.  **Configure Keystore:** Add your keystore file and `gradle.properties` configuration manually.
3.  **Compile:**
    Navigate to the `android` directory and use Gradle directly to output a production binary:
    ```bash
    cd android
    ./gradlew app:assembleRelease # For APKs
    ./gradlew app:bundleRelease   # For App Store AABs
    ```

### iOS Release Build (IPA)
1.  **Generate Native Code:** Run `npx expo prebuild` to create the `/ios` directory.
2.  **Open in Xcode:** 
    ```bash
    xed ios
    ```
3.  **Compile:** Select your signing certificate in Xcode's "Signing & Capabilities", choose "Release" from Product > Scheme > Edit Scheme, and finally go to **Product > Archive** to build the final IPA.

## 8. Phase 7: GitHub Actions CI/CD Automation

To maintain the open-source repository and ensure automated, reliable releases for contributors without relying on external cloud compilers, Ether utilizes **GitHub Actions** for standard open-source CI/CD.

*   **Automated PR Checks:** A GitHub workflow automatically runs ESLint, Prettier, and Jest unit tests on every pull request.
*   **Local Artifact Compilation on Release:** When a new release tag is pushed, a GitHub Action spins up an Ubuntu/macOS runner, executes `npx expo prebuild`, and runs the standard Android `./gradlew assembleRelease` and iOS `xcodebuild` scripts locally on the GitHub runner.
*   **Release Assets:** The resulting `.apk` is automatically attached as an asset to the GitHub Release, allowing users to download and sideload the app directly from the repository completely free of charge.

## 9. Phase 8: Production Monitoring and Error Handling

Since Ether is designed to run headless for weeks or months, we must aggressively monitor its stability in the background using open-source compatible tools.

*   **Crash Reporting (Sentry):** Sentry captures fatal crashes and JavaScript exceptions. By running the Sentry wizard (`npx @sentry/wizard@latest -i reactNative`), the app is configured to automatically catch unhandled errors. We use `npx sentry-expo-upload-sourcemaps dist` during our GitHub Actions pipeline to upload sourcemaps directly to our Sentry instance.

## 10. Phase 9: Performance Optimization & Tooling

To ensure Ether operates smoothly over long periods without memory leaks or thermal throttling:

*   **Hermes Engine:** Ether uses the Hermes engine, which is the default JavaScript engine optimized for React Native. It improves start-up time, decreases binary size, and uses significantly less memory at runtime.
*   **Tree Shaking & Minification:** To optimize the production bundle, tree shaking (`EXPO_UNSTABLE_TREE_SHAKING=1`) is enabled to strip out dead code. The Metro bundler's minification process uses Terser to strip out all `console.log` statements (`drop_console: true`) in production, preventing memory bloat from continuous background logging.
*   **React Compiler:** Adopting the React Compiler via `"reactCompiler": true` within the `experiments` object in `app.json`.

## 11. Phase 10: Application Testing Strategy

Testing heavy native integrations requires a robust strategy.

*   **Unit Testing (Jest):** `jest-expo` is used to write unit tests for the React UI components and state logic.
*   **Mocking Native Modules:** Because the custom foreground service and OpenCV motion detection modules rely on physical hardware and cannot run in a Node.js testing environment, we generate automatic mock implementations of our custom native modules using the `npx expo-modules-test-core generate-ts-mocks` command.

## 12. Phase 11: Native Lifecycle and OS Integration

To reliably run a security app completely headless, Ether must react precisely to OS-level backgrounding and foregrounding events.

*   **Android Lifecycle Listeners:** The custom foreground service module utilizes `ReactActivityLifecycleListener` in Kotlin to hook directly into the Android `Activity` lifecycle (`onCreate`, `onResume`, `onPause`, `onDestroy`). This ensures the app can gracefully pause non-essential UI rendering when pushed to the background.
*   **iOS AppDelegate Subscribers:** For iOS parity, the app utilizes `ExpoAppDelegateSubscriber` to hook into `applicationDidEnterBackground` and `applicationWillEnterForeground`, managing background task constraints seamlessly.

---
