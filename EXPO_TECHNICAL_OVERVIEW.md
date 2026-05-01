# Expo Dev Server, Bundler, and Expo Go Native Preview: Technical Overview

This document summarizes how the Expo dev server, Metro bundler, and Expo Go native app work together to provide a seamless development experience.

## 1. Expo Dev Server and Metro Bundler (`@expo/cli`)

When you run `npx expo start`, the Expo CLI initializes a development server environment managed by `DevServerManager`.

### Metro Initialization
- **`MetroBundlerDevServer`**: This class (in `packages/@expo/cli/src/start/server/metro/MetroBundlerDevServer.ts`) is responsible for managing the Metro instance.
- **`instantiateMetroAsync`**: Configures Metro with Expo-specific settings. It uses `loadMetroConfigAsync` to merge the project's `metro.config.js` with Expo's defaults (from `@expo/metro-config`).
- **Middleware Chain**: Expo injects a custom middleware chain into the Metro server. Key middlewares include:
    - `ExpoGoManifestHandlerMiddleware`: Handles requests for the app manifest.
    - `RuntimeRedirectMiddleware`: Handles deep linking and redirects to the correct runtime (Expo Go or Dev Client).
    - `InterstitialPageMiddleware`: Serves the "loading" or "selection" page.

### Manifest Generation
- **`ExpoGoManifestHandlerMiddleware`**: When Expo Go requests the manifest (usually at `/`), this middleware constructs an **Expo Updates Manifest**.
- **Content**: The manifest includes:
    - `launchAsset`: A URL pointing to the JS bundle (e.g., `http://192.168.1.1:8081/index.bundle?platform=ios&dev=true&hot=false`).
    - `extra.expoClient`: The project's configuration (from `app.json` or `app.config.js`).
    - `extra.expoGo`: Runtime-specific settings like the debugger host and main module name.

## 2. Expo Go Native Loading (iOS and Android)

Expo Go acts as a "universal runner" that can host different React Native experiences.

### Initialization (The "Kernel")
Both platforms use a "Kernel" to manage the lifecycle of applications.
- **iOS (`EXKernel`)**: Manages `EXKernelAppRecord` instances.
- **Android (`Kernel.kt`)**: Manages `ExperienceActivity` tasks.

### App Loading Process
1.  **Manifest Fetching**:
    - **iOS**: `EXAppLoaderExpoUpdates` uses `EXUpdatesAppLoaderTask` to fetch the manifest from the dev server. It handles the `multipart/mixed` response which may include code-signing certificates.
    - **Android**: `ExpoUpdatesAppLoader` performs a similar task, fetching and parsing the manifest.
2.  **JS Bundle Loading**:
    - Once the manifest is received, the app identifies the `launchAsset` URL.
    - In development mode, Expo Go bypasses the standard updates caching and fetches the JS bundle directly from the Metro server via the `bundleUrl` specified in the manifest.
3.  **Runtime Instantiation**:
    - **iOS**: A `RCTBridge` (or `RCTHost` in newer versions) is created using the fetched JS bundle.
    - **Android**: A `ReactInstanceManager` is built, configured with the JS bundle file path and native modules (provided by `ExponentPackage`).
4.  **Native Module Integration**:
    - Expo Go provides a vast array of native modules (the Expo SDK) pre-linked into the binary. These are made available to the React Native guest app during bridge initialization.

## 3. The End-to-End Flow

1.  **CLI**: `npx expo start` starts Metro on port 8081 and waits for connections.
2.  **Handshake**: The user opens Expo Go and enters the URL (or scans a QR code).
3.  **Request**: Expo Go sends an HTTP request to the CLI (with headers like `expo-platform: ios`).
4.  **Response**: The CLI's `ExpoGoManifestHandlerMiddleware` generates and returns a JSON manifest.
5.  **Fetch**: Expo Go reads the `launchAsset.url` from the manifest and requests the JS bundle from Metro.
6.  **Bundle**: Metro bundles the project's JavaScript on-the-fly and streams it to the device.
7.  **Run**: Expo Go initializes the React Native environment with the received bundle and renders the root component.

## Summary of Key Components

| Component | Path (Representative) | Role |
| :--- | :--- | :--- |
| **Dev Server Manager** | `packages/@expo/cli/src/start/server/DevServerManager.ts` | Orchestrates multiple bundlers (Metro, Webpack). |
| **Metro Dev Server** | `packages/@expo/cli/src/start/server/metro/MetroBundlerDevServer.ts` | Wraps Metro and attaches Expo middleware. |
| **Manifest Middleware** | `packages/@expo/cli/src/start/server/middleware/ExpoGoManifestHandlerMiddleware.ts` | Generates the `expo-updates` manifest. |
| **iOS App Loader** | `apps/expo-go/ios/Exponent/Kernel/AppLoader/EXAppLoaderExpoUpdates.m` | Fetches manifest/bundle on iOS. |
| **Android Kernel** | `apps/expo-go/android/expoview/src/main/java/host/exp/exponent/kernel/Kernel.kt` | Manages app tasks and runtimes on Android. |
