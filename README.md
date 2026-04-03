# Velocity

A minimalist first-person tunnel racer for Android, inspired by SpeedX 3D.

## Prerequisites

- **Java 17+** — bundled with Android Studio at `<Android Studio>/jbr/`
- **Android SDK** — installed via Android Studio (platform 36, build-tools)
- **Gradle** — the project includes a Gradle wrapper (`gradlew`), so no global install is needed

Set these environment variables (or rely on `local.properties` for the SDK):

```sh
# PowerShell example — adjust paths to your system
$env:JAVA_HOME = "C:\Users\<you>\AppData\Local\Programs\Android Studio\jbr"
$env:ANDROID_HOME = "C:\Users\<you>\AppData\Local\Android\Sdk"
```

The project includes a `local.properties` file (git-ignored) that points to the SDK. If it's missing, create it:

```
sdk.dir=C:\\Users\\<you>\\AppData\\Local\\Android\\Sdk
```

## Build

```sh
# Debug APK
./gradlew assembleDebug

# Release APK (minified)
./gradlew assembleRelease
```

Output: `app/build/outputs/apk/debug/app-debug.apk`

## Run on Emulator

### 1. Create an emulator (one-time)

Open Android Studio → **Device Manager** → **Create Virtual Device** → pick any phone (e.g. Pixel 6) → select a system image (API 36) → Finish.

Or via command line:

```sh
# List available system images
sdkmanager --list | Select-String "system-images"

# Install a system image (if not already present)
sdkmanager "system-images;android-36;google_apis;x86_64"

# Create an AVD
avdmanager create avd -n Pixel6_API36 -k "system-images;android-36;google_apis;x86_64" -d "pixel_6"
```

### 2. Launch the emulator

```sh
# From Android Studio: Device Manager → ▶ Play button
# Or from command line:
emulator -avd Pixel6_API36
```

### 3. Install & run

```sh
# Build + install + launch in one step
./gradlew installDebug

# Then open the app on the emulator — it's called "Velocity"
```

Or manually with adb:

```sh
adb install app/build/outputs/apk/debug/app-debug.apk
adb shell am start -n com.velocity/.VelocityActivity
```

## Run on Physical Device

1. **Enable Developer Options** on your phone: Settings → About Phone → tap "Build Number" 7 times.
2. **Enable USB Debugging**: Settings → Developer Options → USB Debugging → On.
3. Connect the phone via USB and accept the debugging prompt.
4. Verify the device is visible:

```sh
adb devices
```

5. Install and launch:

```sh
./gradlew installDebug
# App appears as "Velocity" in the launcher

# Or manually:
adb install app/build/outputs/apk/debug/app-debug.apk
adb shell am start -n com.velocity/.VelocityActivity
```

> **Tip:** `adb`, `sdkmanager`, `avdmanager`, and `emulator` live inside the Android SDK.
> Add these to your PATH for convenience:
> ```
> $env:ANDROID_HOME/platform-tools     (adb)
> $env:ANDROID_HOME/cmdline-tools/latest/bin  (sdkmanager, avdmanager)
> $env:ANDROID_HOME/emulator           (emulator)
> ```

## Docs

- [Spec](doc/spec.md)
- [Architecture](doc/architecture.md)
- [Backlog](doc/backlog.md)
