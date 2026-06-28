# Detailed Technical Specification: Open-Source Local DVR

## 1. Introduction

This document provides a comprehensive technical blueprint for the open-source, local-first DVR security camera application. It expands upon the initial high-level specification by providing in-depth implementation details, UI/UX design principles, and a phased development plan. The core mission is to create a privacy-centric security tool by upcycling old mobile devices into powerful, standalone surveillance units that do not rely on cloud infrastructure.

This specification is a living document intended to guide developers through the entire lifecycle of the project, from environment setup to final deployment.

---

## 2. Architectural Strategy

The application's core requirement is deep integration with hardware (camera, filesystem) and system-level services (background execution), which surpasses the capabilities of standard React Native or basic Expo Go projects. To address this, the architecture is founded on the **Expo framework**, leveraging two key concepts for full native control: **Continuous Native Generation (CNG)** and **Development Builds**.

*   **Rejection of Expo Go:** The standard Expo Go client is a pre-compiled binary that cannot be modified with custom native code. Because our app demands persistent background camera access and custom C++ bindings for motion detection, Expo Go is not a viable solution.

*   **Development Builds (`expo-dev-client`):** We will use `expo-dev-client` to compile the native Android and iOS projects locally. This creates a custom version of the Expo client that includes our own native modules, giving us full control over the hardware and OS while retaining the rapid refresh and debugging capabilities of the Expo ecosystem.

*   **Continuous Native Generation (CNG):** The native `/android` and `/ios` project directories are not meant to be heavily modified directly. Instead, they are treated as build artifacts generated on-demand by Expo's build tools based on the configuration in `app.json` and `package.json`. Native dependencies and permissions are managed through **Config Plugins**, which programmatically modify the native project files at build time. This approach allows the team to focus on a clean JavaScript/TypeScript codebase while ensuring native projects are reproducible and less prone to configuration drift.

*   **Expo Modules API & JSI:** All custom native features (background services, storage management, OpenCV integration) will be developed as standalone packages using the modern **Expo Modules API**. This API provides an idiomatic and type-safe DSL for both Swift (iOS) and Kotlin (Android). Crucially, it is heavily optimized to use React Native's **JavaScript Interface (JSI)**, allowing for high-performance, direct, and synchronous communication between the JavaScript and native threads, which is essential for real-time camera frame processing.

---

## 3. UI/UX Designs

The user experience is designed around a "set and forget" philosophy. The interface must be highly readable and intuitive for a broad audience, guiding them from initial setup to long-term headless operation with minimal friction. The primary design language will be clean and functional, using a card-based layout and clear, geometric typography.

### 3.1. Theming and Style Guide

To ensure a consistent and maintainable visual identity, the application will enforce a strict theming strategy. All UI components that involve color must adhere to the centralized theming system.

*   **Single Source of Truth:** The `src/constants/theme.ts` file is the single source of truth for all design tokens, including colors, fonts, and spacing.
*   **Dynamic Colors with `useColors`:** All components **must** use the `useColors` hook (from `src/hooks/use-colors.ts`) to access the appropriate color palette. This hook automatically returns the correct set of colors based on the user's selected color scheme (light or dark).
*   **No Hardcoded Colors:** Hardcoding color values directly in component styles (e.g., `color: '#FFF'`, `backgroundColor: 'black'`) is strictly forbidden. Instead, use the properties from the object returned by `useColors`.

**Example Usage:**

```tsx
import { Text, View, StyleSheet } from 'react-native';
import { useColors } from '@/hooks/use-colors';
import { Spacing } from '@/constants/theme';

const ThemedComponent = () => {
  const colors = useColors();

  return (
    <View style={[styles.container, { backgroundColor: colors.background }]}>
      <Text style={[styles.text, { color: colors.text }]}>
        This text will be black in light mode and white in dark mode.
      </Text>
    </View>
  );
};

const styles = StyleSheet.create({
  container: {
    flex: 1,
    padding: Spacing.three,
  },
  text: {
    fontSize: 16,
  },
});
```

### 3.2. Main Dashboard (Home Screen)

This is the user's primary entry point, acting as a simple and clear launchpad.

*   **Layout:** A two-card layout.
*   **Primary Action Card:** A large, high-contrast card with a button labeled **"Start Security Camera"**. This is the main call-to-action.
*   **Navigation Cards:** Smaller, secondary cards providing quick access to:
    *   **Gallery:** Shows a small thumbnail of the latest recording and a badge with the count of unwatched events.
    *   **Settings:** An icon-based link to the main settings page.
    *   **System Status:** An icon-based link to the activity logs page.

### 3.3. Camera Setup Mode

This mode is entered after tapping the primary action card. It allows the user to physically position the device while viewing a live camera feed.

*   **Live Configuration Overlay:** A non-intrusive overlay on top of the camera feed provides access to critical hardware settings:
    *   **Camera Toggle:** Icon button to switch between front and rear cameras.
    *   **Flash Toggle:** Icon button to enable the torch.
    *   **Orientation Lock:** Icon button to lock the screen in portrait or landscape.
    *   **Motion Preview:** The border of the screen will flash or change color subtly to provide real-time visual feedback when the OpenCV module detects motion, helping the user calibrate sensitivity.
*   **Finalization Button:** A clearly labeled button, **"Activate Power Saving Mode,"** which initiates the final lockdown state.

### 3.4. Power Saving "Stealth" Mode

This is the default, long-term operational state of the application. The goal is to make the device appear as if it is turned off, preventing screen burn-in and reducing thermal load.

*   **Visuals:** The screen will be completely black. On OLED/AMOLED screens, this turns the pixels off entirely.
*   **System UI:** The app will enter Android's "Immersive Mode," hiding the top status bar and bottom navigation bar.
*   **Brightness:** Screen brightness will be programmatically set to the lowest possible value, and the "Extra Dim" system flag will be applied if available.
*   **Input:** All touch input on the screen will be disabled to prevent accidental interactions.
*   **Exit Condition:** The only way to exit this mode is by pressing the device's physical **Power Button**, which will return the user to the lock screen or the app's dashboard.

### 3.5. Internal Gallery and Settings

*   **Gallery:** A vertically scrolling grid of thumbnails, sorted chronologically. Each item will display the timestamp and duration of the recording. A long-press action will reveal a context menu with options to **Delete**, **Share**, or **Favorite** the clip (up to the capped limit of 10).
*   **Settings:** A simple, list-based screen with toggles and sliders for configuring:
    *   Motion Detection (On/Off)
    *   Motion Sensitivity (Slider)
    *   Storage Allocation (Slider or text input for GB limit)

---

## 4. Development Phases & Implementation Details

This section provides a granular, step-by-step guide for implementing the application. Each phase is designed to be a self-contained unit of work.

### Phase 1: Project Scaffolding & Development Environment

This phase establishes the project's foundation.

**1.1. System Prerequisites**
Ensure your development environment is configured for React Native development.
- **Reference:** [React Native Development Environment Setup](https://reactnative.dev/docs/environment-setup)

**1.2. Expo Project Initialization**
Bootstrap a new Expo project with the TypeScript template.

```bash
npx create-expo-app@latest GroundhogLens --template blank-ts
cd GroundhogLens
```

**1.3. Development Client Setup**
The app requires custom native code, so we must use a development build instead of Expo Go.
- **Reference:** [Expo Development Builds](https://docs.expo.dev/develop/development-builds/introduction/)

1.  Install the `expo-dev-client` library:
    ```bash
    npx expo install expo-dev-client
    ```

**1.4. First Native Build & Verification**
Compile the app locally to create the development client on a simulator or physical device.

- For iOS:
  ```bash
  npx expo run:ios
  ```
- For Android:
  ```bash
  npx expo run:android
  ```

### Phase 2: Application UI & Navigation

This phase builds the static UI shell using Expo Router.

**2.1. Installing Expo Router**
- **Reference:** [Expo Router Installation](https://docs.expo.dev/router/installation/)

```bash
npx expo install expo-router react-native-safe-area-context react-native-screens expo-linking expo-constants expo-status-bar
```

**2.2. Configure Project**
1.  Set up the entry point by modifying `package.json`:
    ```json
    "main": "expo-router/entry"
    ```
2.  Add the `expo-router` scheme to `app.json` under the `expo` key:
    ```json
    "scheme": "groundhoglens"
    ```

**2.3. Create File-Based Route Structure**
Create the following directory and files inside the `app` directory:
- `app/_layout.tsx`: The root layout.
- `app/index.tsx`: The home/dashboard screen.
- `app/gallery.tsx`: The gallery screen.
- `app/settings.tsx`: The settings screen.
- `app/status.tsx`: The system status screen.

### Phase 3: Core Engine - Native Module Implementation

This phase involves writing the three critical custom native modules.

**3.1. Scaffolding a Local Native Module**
- **Reference:** [Creating a new module](https://docs.expo.dev/modules/create-a-new-module/)

For each of the modules below, you will first run:
```bash
npx create-expo-module@latest --local
```
Then, follow the prompts to name your module (e.g., `react-native-camera-foreground-service`).

**3.2. Module 1: `react-native-camera-foreground-service`**

*   **Purpose:** To keep the app alive and authorized for camera use in the background.
*   **Android Implementation (Kotlin):**
    1.  **Add Permissions:** In the module's `AndroidManifest.xml`, add:
        ```xml
        <uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
        <uses-permission android:name="android.permission.FOREGROUND_SERVICE_CAMERA" />
        ```
    2.  **Implement the Service:** In the module's Kotlin file, create a method to start a `ForegroundService`. This involves creating a `NotificationChannel` and a `Notification`, then calling `context.startForegroundService(intent)`.
        - **Reference:** [Android Foreground Services](https://developer.android.com/guide/components/foreground-services)
*   **iOS Implementation (Swift):**
    1.  **Configure Background Modes:** In the project's `Info.plist`, add "App plays audio" (`audio`) to the `UIBackgroundModes` array. This is a common technique to request extended background execution time.
    2.  **Manage Audio Session:** Use `AVAudioSession` to start a silent audio track when the app enters the background to maintain its active state.
        - **Reference:** [Playing Background Audio](https://developer.apple.com/documentation/avfoundation/media_playback_and_selection/playing_audio_in_the_background)
*   **JavaScript API Definition:** The module's `*Module.swift`/`.kt` file will expose:
    - `startCameraService(notificationTitle: string, notificationBody: string): Promise<void>`
    - `stopCameraService(): Promise<void>`
    - `isServiceActive(): Promise<boolean>`

**3.3. Module 2: `react-native-directory-limit-manager`**

*   **Purpose:** To perform high-performance, native-level FIFO deletion of files in a directory.
*   **Android Implementation (Kotlin):**
    1.  Use `java.io.File` to list files in the provided directory path.
    2.  Sort the files by `lastModified()`.
    3.  Iterate through the sorted list, deleting files until the directory size is under the specified limit, respecting the `exclude` list.
*   **iOS Implementation (Swift):**
    1.  Use `FileManager.default` to get the contents of the directory.
    2.  Retrieve the `creationDate` attribute for each file.
    3.  Sort the files and perform deletion, respecting the `exclude` list.
*   **JavaScript API Definition:**
    - `enforceLimit(options: { directory: string; maxSize: number; exclude?: string[] }): Promise<void>`
    - `getDirectorySize(directory: string): Promise<number>`

**3.4. Module 3: `react-native-opencv-motion-detector`**

*   **Purpose:** A C++ JSI worklet for real-time motion detection.
*   **Dependencies:** This requires `react-native-vision-camera` and a pre-compiled OpenCV library.
    - Vision Camera: `npx expo install react-native-vision-camera`
    - OpenCV: Will need to be added to the module's native build files.
*   **Android C++/CMake Configuration:**
    1.  Add the OpenCV dependency to the module's `build.gradle`.
    2.  Configure `CMakeLists.txt` to find and link the OpenCV libraries.
*   **C++ Frame Processor Logic:**
    - Create a C++ function that takes a `Frame` object from Vision Camera.
    - Use `cv::Mat` to represent the frame data (isolating the Y channel).
    - Use `cv::absdiff`, `cv::threshold`, and `cv::countNonZero` to perform the motion analysis.
    - **Reference:** [Vision Camera Frame Processors](https://react-native-vision-camera.com/docs/guides/frame-processors)
*   **Integration with Vision Camera:** The module will export a function that can be used with Vision Camera's `useFrameProcessor` hook.

### Phase 4: Integrating the Core Engine

This phase connects the UI to the native modules.

**4.1. Camera View Implementation**
- Install `react-native-vision-camera`.
- In your main camera screen component, request camera permissions and render the `<Camera>` component, ensuring it is active.

**4.2. Connecting Motion Detection**
- Import the motion detection worklet from your native module.
- Use the `useFrameProcessor` hook to run the detection logic on every frame.

```tsx
import { useFrameProcessor } from 'react-native-vision-camera';
import { detectMotion } from 'react-native-opencv-motion-detector';

// ...
const frameProcessor = useFrameProcessor((frame) => {
  'worklet';
  const isMotion = detectMotion(frame, sensitivity);
  if (isMotion) {
    console.log('Motion Detected!');
    // Trigger recording logic here
  }
}, [sensitivity]);
```

**4.3. Triggering Video Recording**
- When the frame processor detects motion, call the `startRecording` method on the camera's `ref`.
- **Reference:** [Vision Camera `startRecording`](https://react-native-vision-camera.com/docs/api/classes/Camera#startrecording)

**4.4. Managing Storage**
- After a video chunk is recorded, call the `enforceLimit` function from `react-native-directory-limit-manager` to clean up old files asynchronously.

---

## 5. Use Case Classifications

The application is designed to be flexible, serving several distinct user needs.

### 5.1. Primary Use Case: Set-and-Forget Home Security

This is the core user story the application is built for.

*   **User Profile:** A homeowner, renter, or small business owner who wants a simple, no-cost security solution for a single room or entryway. They are privacy-conscious and prefer to keep their data off the cloud.
*   **Environment:** The user has a spare or older smartphone that can be dedicated to this single purpose. The device will be plugged into a constant power source and placed in a fixed position (e.g., on a bookshelf, mounted to a wall).
*   **User Journey:**
    1.  The user installs the app on their old phone.
    2.  They walk through the initial setup, positioning the camera and activating the "Power Saving Mode."
    3.  The device is left untouched for days or weeks, running continuously.
    4.  The user only physically interacts with the device to periodically review the motion-capture events in the internal gallery.

### 5.2. Secondary Use Case: Temporary Monitoring

This use case leverages the app's quick setup for short-term, ad-hoc monitoring needs.

*   **User Profile:** A parent, pet owner, or caregiver.
*   **Environment:** The user needs to monitor a space for a limited duration (e.g., a few hours). The device might be running on battery power.
*   **User Journey:**
    1.  The user sets up the phone to monitor a baby's crib during a nap, or to watch a pet while they are away for an afternoon.
    2.  They activate the camera but may not use the full "Power Saving Mode" if they intend to retrieve the phone shortly.
    3.  After the period is over, they stop the camera service and review the recorded clips for any events of interest. The recordings may be deleted immediately after viewing.

### 5.3. Secondary Use Case: Developer/Tinkerer Playground

This use case targets the open-source community.

*   **User Profile:** A React Native developer, native mobile developer, or hardware enthusiast.
*   **Environment:** A development machine where they can clone the repository, install dependencies, and run the app in a debug environment.
*   **User Journey:**
    1.  The developer clones the project to experiment with the custom native modules.
    2.  They might fork the `react-native-opencv-motion-detector` module to experiment with different OpenCV algorithms or to integrate a different C++ library.
    3.  They could use the `react-native-directory-limit-manager` as a standalone package in their own, completely unrelated application.
    4.  The app serves as a real-world example of how to integrate complex native code within the Expo ecosystem.

---

## 6. Future Scope & Hybrid AI Pipeline

While the MVP is focused on creating a robust local DVR, the architecture is designed to support significant future enhancements.

### 6.1. Hybrid AI Motion Pipeline (Post-MVP)

The MVP's reliance on pixel-differencing is highly efficient but prone to false positives from environmental changes (e.g., shifting shadows, flickering lights). A hybrid AI pipeline will be implemented to drastically improve accuracy without sacrificing battery life.

*   **Concept:** Use the hyper-efficient C++ OpenCV worklet as a low-power "wake-up" call. The ML model remains dormant until the C++ layer detects a potential motion event.
*   **Workflow:**
    1.  **Stage 1 (Always On):** The `react-native-opencv-motion-detector` runs continuously, as in the MVP.
    2.  **Stage 2 (Wake on Trigger):** When the C++ worklet detects a significant pixel change, instead of immediately recording, it activates a second Frame Processor.
    3.  **Stage 3 (Validation):** This second Frame Processor runs a lightweight, quantized TensorFlow Lite (TFLite) object detection model (e.g., MobileNetV3-SSD) for a few frames.
    4.  **Stage 4 (Confirmation):** If the TFLite model detects a specific object of interest (e.g., "person," "car") with a high confidence score (>80%), the app confirms the motion and begins recording. If no object is detected, the event is dismissed as a false positive, and the TFLite Frame Processor goes back to sleep.
*   **Benefit:** This approach provides the high accuracy of machine learning without the severe battery and thermal cost of running an ML model continuously. The expensive inference only runs for a few seconds per potential event.
*   **Implementation:**
    - A TFLite runtime library for React Native (e.g., `react-native-fast-tflite`) will be integrated.
    - The `useFrameProcessor` logic will be updated to chain the two worklets.

### 6.2. Peer-to-Peer (P2P) Live Streaming (Post-MVP)

This feature introduces the ability to view the camera's live feed from another device remotely, without routing video through a central server.

*   **Technology:** WebRTC (`react-native-webrtc`)
*   **Architecture:**
    1.  **Signaling:** A lightweight signaling server will be required to broker the initial connection between the two devices (the camera and the viewer). This server does *not* touch the video stream; it only exchanges metadata (like IP addresses and stream information) to help the two peers find each other. This can be a simple WebSocket server.
    2.  **ICE (STUN/TURN):** To handle network address translation (NAT) and firewalls, STUN/TURN servers will be needed. For the open-source version, the app will be configured to use public STUN servers and will allow users to provide their own TURN server credentials if needed.
    3.  **Connection:** Once the connection is established, video and audio are streamed directly from the camera device to the viewer device, encrypted end-to-end via DTLS, which is built into WebRTC.
*   **User Experience:** The viewer device will pair with the camera device by scanning a QR code, which securely transmits the initial signaling server information and authentication keys.

### 6.3. Monetization Strategy (Post-MVP)

While the core local DVR functionality will remain free and open-source, a premium tier can be introduced to fund the project's maintenance and the hosting of TURN servers.

*   **Freemium Model:**
    - **Free Tier:** Includes all local recording features. Users who want remote streaming can self-host their own signaling/TURN servers.
    - **Premium Tier (Subscription):** A monthly subscription could provide:
        - **Managed TURN Server Access:** "Zero-config" remote streaming without requiring users to set up their own servers.
        - **Higher Resolution Streaming:** Access to 1080p or 4K remote streaming.
        - **Cloud Backup (Optional):** An option for users to back up their 10 "favorite" clips to a private cloud storage bucket. This would be an opt-in feature, respecting the project's privacy-first ethos.
