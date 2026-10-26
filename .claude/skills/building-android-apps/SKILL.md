---
name: building-android-apps
description: Build, sign and install the Android side of a MobAI App Builder app through MobAI's local HTTP API, on macOS, Windows or Linux. Covers debug builds and signed release App Bundles with Gradle for Flutter, React Native, Expo and native projects, the upload keystore App Builder keeps in the keychain, versionCode, installing a debug build on a claimed emulator or phone, and fixing the Gradle failures that come up most (JDK, SDK path, AGP and Gradle versions, R8, 16 KB pages). Use when asked to build the Android app, make an APK or App Bundle, sign a release, set up or back up a keystore, fix a Gradle build, "assembleDebug failed", "bundleRelease", "install it on the emulator", or anything about Android build and signing in App Builder.
---

# Building and signing Android apps

App Builder builds Android with Gradle on this machine, whatever its OS, and
installs over adb. You drive it over MobAI's HTTP API:

```sh
API="${MOBAI_API_URL:-http://127.0.0.1:8686}/api/v1"
APP="$API/apps/$MOBAI_APP_ID"
curl -s "$APP"      # android.kinds: "debug", plus "release" once a keystore exists
```

Errors come back as `{"error": "..."}` with App Builder's own sentence.

## What a build runs

| Project | Debug | Release |
|---|---|---|
| Flutter (`pubspec.yaml`) | `flutter build apk --debug` | `flutter build appbundle --release` |
| Expo | JS install if `node_modules` is missing, `npx expo prebuild --platform android` if there is no `android/` project, then `./gradlew assembleDebug` | same prep, `./gradlew bundleRelease` |
| React Native | JS install if `node_modules` is missing, `android/gradlew assembleDebug` | `bundleRelease` |
| Native | `./gradlew assembleDebug` (`gradlew.bat` on Windows) | `bundleRelease` |

The project's own Gradle wrapper is used; a missing wrapper falls back to
`gradle` on PATH. The JS install uses the manager the lockfile names (pnpm,
yarn, bun, npm). A Flutter project that pins a version with fvm (`.fvmrc`
in its folder or above) builds with that version; the build fails if fvm
has not installed it, so run `fvm install <version>` rather than removing
the pin.

## Debug build, then install

```sh
curl -s -X POST "$APP/builds" -d '{"platform":"android","kind":"debug"}'
curl -s "$APP/builds/active"     # poll every 15 to 30 s until phase is done or failed
curl -s "$APP/builds"            # newest first; take its id (or clientBuildId)
```

Claim the device, then install (the `controlling-mobile-devices` skill
covers driving it afterwards):

```sh
curl -s -X POST "$API/devices/claim" -d "{\"device\":\"<device id>\",\"owner\":\"$MOBAI_CLAIM_OWNER\"}"
curl -s -X POST "$APP/builds/<build id>/install" -H "X-Lease-Token: <leaseToken>" \
  -d '{"device_id":"<device id>"}'
```

For quick iteration on Flutter, Expo and React Native, hot reload on a
running debug build beats a rebuild; build through App Builder when the
user wants the build recorded, or before a release.

## Release: the upload keystore

A release is a signed App Bundle (`.aab`) for Google Play. App Builder
signs it with the App's upload keystore, which the user makes or imports
once in App Builder's Google Play dialog. The passwords stay in the OS
keychain and reach Gradle as Android's injected signing properties, so:

- Do not add a `signingConfigs` block with passwords to `build.gradle`, and
  do not put passwords in `gradle.properties`, `local.properties` or an env
  file. A project that already has a release `signingConfig` keeps working;
  the injected properties take its place for App Builder's build.
- Do not commit the keystore file. Remind the user to back it up off this
  machine. With Play App Signing (the default) Google holds the real app
  signing key and the upload key can be reset, but the reset takes days.
- `android.kinds` without `release` means there is no keystore yet. Only
  the user can add it: they open the Google Play dialog.

```sh
curl -s -X POST "$APP/builds" -d '{"platform":"android","kind":"release"}'
```

A release bundle does not install on a device; check the same commit with a
debug build first. Uploading to Google Play is the `releasing-mobile-apps`
skill.

## versionCode and versionName

Google Play refuses a `versionCode` it already has, and App Builder does not
bump it. Before a release, raise it:

- Flutter: the number after `+` in `pubspec.yaml` `version: 1.4.0+27`
- Expo: `expo.android.versionCode` and `expo.version` in `app.json`
  (prebuild writes them into `android/`; edit `app.json`, not the
  generated files, when `android/` is not in git)
- React Native and native: `versionCode` and `versionName` in
  `android/app/build.gradle` (or `app/build.gradle.kts`)

`versionCode` is an integer that only grows. Keep `versionName` equal to
the iOS marketing version when the App ships on both.

## Gradle failures

A failed build has `build.logPath`, in the `failed` event and in
`GET $APP/builds`: a file with the log's last 40 lines. Read it first. The
usual causes:

| Log says | Fix |
|---|---|
| `Unsupported class file major version`, `requires Java 17` | JDK mismatch: AGP 8 needs JDK 17 or newer. Point `JAVA_HOME` at Android Studio's bundled JBR, or set `org.gradle.java.home` in the user's `~/.gradle/gradle.properties`, then restart MobAI so it sees the change |
| `SDK location not found` | `ANDROID_HOME` unset and no `sdk.dir` in `local.properties`: write `sdk.dir` there (never commit it) |
| `Minimum supported Gradle version is` | the wrapper is older than the Android Gradle plugin wants: raise `distributionUrl` in `gradle/wrapper/gradle-wrapper.properties` to the version named |
| `Failed to install the following Android SDK packages`, licences not accepted | the user runs `sdkmanager --licenses` once, or opens the SDK Manager in Android Studio |
| `Duplicate class`, `checkDebugAarMetadata` | dependency versions disagree; align them, or raise `compileSdk` to what the AAR metadata asks for |
| release only: `ClassNotFoundException` at runtime, missing JSON fields | R8 removed classes used by reflection: add `-keep` rules to `proguard-rules.pro` for the models and native bridges |
| `no gradle wrapper ... and no gradle on PATH` | the project has no wrapper: run `gradle wrapper` once if Gradle is installed, or restore `gradlew` from the template |
| `flutter is not on PATH`, `npx is not on PATH` | the user installs Flutter or Node.js where MobAI can see it |
| `a build of this app is already running` | wait for `builds/active` to reach `done` or `failed` |

Before a release, also check what Google Play enforces at upload: `targetSdk`
at the level Play Console names, and native libraries built for 16 KB
memory pages (a current NDK and current versions of native SDKs).

## Do not

- Do not run `./gradlew` by hand to "help" a build App Builder is running;
  two Gradle builds of one project fight over the same outputs.
- Do not change `applicationId` to make an upload work. It is the app's
  identity on Google Play and cannot change after the first release.
- Do not delete `android/` in an Expo project that keeps it in git; it may
  hold native changes prebuild would not recreate.
