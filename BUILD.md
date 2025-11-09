# Building MPV Remote 2 APK

## Build Instructions

Follow these steps to build the Android APK:

```bash
# 1. Build the Vue app
cd /home/alarm/projects/mpv-remote-app
npm run build

# 2. Copy web assets to Android (use 'copy' instead of 'sync' to avoid re-downloading Gradle dependencies)
npx cap copy android

# 3. Build the APK
cd android
./gradlew clean assembleDebug

# 4. Copy the APK to home directory
cp app/build/outputs/apk/debug/app-debug.apk ~/mpv-remote-2.apk
```

## Important Notes

- **Use `npx cap copy android` instead of `npx cap sync android`** to avoid re-downloading x86 build tools that don't work on ARM64
- If you accidentally run `npx cap sync android`, you'll need to replace the build tools again with ARM64 versions:
  ```bash
  for tool in aapt aapt2 aidl dexdump split-select zipalign; do
    find ~/.gradle/caches -name "$tool" -type f -exec cp ~/build-tools/$tool {} \;
  done
  ```

## Quick Build (after first successful build)

If you've already built once and only changed code:

```bash
cd /home/alarm/projects/mpv-remote-app
npm run build
npx cap copy android
cd android
./gradlew assembleDebug
cp app/build/outputs/apk/debug/app-debug.apk ~/mpv-remote-2.apk
```

(Skip the `clean` step for faster builds)

## Output

The final APK will be at: `~/mpv-remote-2.apk`
