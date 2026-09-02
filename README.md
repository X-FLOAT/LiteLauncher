# Lite Launcher

An ultra-lightweight Android home screen launcher with a swipe-up, animated app drawer.

## Why it's "ultra-lite"
- No third-party UI libraries (no image-loading libs, no animation libs) — just AndroidX core + RecyclerView + coroutines.
- App list is queried and icons decoded off the main thread (`AppRepository`), so the UI never stutters loading it.
- Entrance animations use plain `View.animate()` (hardware-accelerated `ViewPropertyAnimator`), not heavier frameworks like Lottie or MotionLayout.
- The drawer is a translucent `Activity`, not a persistent always-inflated view — nothing is drawn until you open it.

## How the pieces fit
- `HomeActivity` — the actual home screen. Registered with `HOME` + `LAUNCHER` intent filters in the manifest, which is what lets Android offer it as a launcher choice. Detects an upward swipe (or a tap on the handle) to open the drawer.
- `AppDrawerActivity` — translucent activity that slides a sheet up from the bottom over a dimmed scrim, using a Material-style decelerate curve. Tap the scrim or press back to slide it back down.
- `AppAdapter` — RecyclerView adapter for the icon grid, with a subtle staggered fade+rise-in animation per icon as it appears (capped delay so long app lists don't feel slow).
- `AppRepository` — loads installed apps via `PackageManager` on a background dispatcher.

## Compatibility
`minSdk` is **24** (Android 7.0 Nougat) — the app now installs and runs on 7.0/7.1 devices, not just 26+. The adaptive launcher icon (`mipmap-anydpi-v26/ic_launcher.xml`) only applies on API 26+; devices below that fall back to the flat vector icon at `res/mipmap/ic_launcher.xml` automatically (no code changes needed elsewhere — nothing else in the source used an API 26+ call).

## Building from the command line (no Android Studio required)
This project already ships a Gradle wrapper (`gradlew` / `gradlew.bat` / `gradle/wrapper/`), so you only need a JDK and the Android SDK command-line tools — not the Android Studio IDE.

1. **Install a JDK 17** (Temurin/Adoptium works well). Confirm with `java -version`.
2. **Get the Android SDK command-line tools** (no IDE): download "Command line tools only" from https://developer.android.com/studio#command-tools, unzip it, and lay it out as:
   ```
   <sdk-root>/cmdline-tools/latest/bin/sdkmanager
   ```
3. **Install the required SDK packages**:
   ```bash
   export ANDROID_HOME=<sdk-root>
   export PATH="$ANDROID_HOME/cmdline-tools/latest/bin:$ANDROID_HOME/platform-tools:$PATH"
   yes | sdkmanager --licenses
   sdkmanager "platform-tools" "platforms;android-34" "build-tools;34.0.0"
   ```
4. **Point Gradle at the SDK** — create `local.properties` in the project root (this file is machine-specific, don't commit it):
   ```
   sdk.dir=/path/to/your/android-sdk
   ```
   (On Windows use `sdk.dir=C\:\\path\\to\\android-sdk`, or just set the `ANDROID_HOME` environment variable instead — Gradle will pick it up.)
5. **Build the APK**:
   ```bash
   ./gradlew assembleDebug        # macOS/Linux
   gradlew.bat assembleDebug      # Windows
   ```
   The output APK lands at `app/build/outputs/apk/debug/app-debug.apk`.
6. **Install it** on a connected device/emulator with adb:
   ```bash
   adb install -r app/build/outputs/apk/debug/app-debug.apk
   ```
7. To actually use it as your launcher: after installing, press the device Home button — Android will ask you to choose a default launcher, or you can set it in Settings → Apps → Default apps → Home app.

For a signed release build instead of debug, run `./gradlew assembleRelease` (you'll need to configure a signing config in `app/build.gradle.kts` first, since this project doesn't ship one).

## Easy things to customize next
- **Icon size / grid columns**: `GridLayoutManager(this, 4)` in `AppDrawerActivity` — change `4` to `5` for smaller icons.
- **Colors / theme**: `res/values/colors.xml` — currently a dark/purple "premium" palette.
- **Wallpaper**: `HomeActivity` currently shows a flat background + clock only (by design, to stay light) — you can read the system wallpaper via `WallpaperManager` and set it as the root background if you want it to look like a real home screen.
- **App icon**: swap `ic_launcher_foreground.xml` for your own vector or PNG mipmaps.
- **Swipe sensitivity**: the fling thresholds (`deltaY > 120`, `velocityY < -600`) are in `HomeActivity.onFling`.

## Known limitations (intentionally left out to keep v1 lite)
- No home-screen widgets support.
- No icon long-press menu (uninstall/app info shortcuts) yet — easy to add via `pm.getLaunchIntentForPackage` + a `PopupMenu`.
- No persisted custom app order/folders — grid is always alphabetical.
