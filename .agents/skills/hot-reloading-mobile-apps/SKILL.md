---
name: hot-reloading-mobile-apps
description: Run a Flutter or React Native app with hot reload on a device through MobAI's local HTTP API, including an iPhone from Windows or Linux with no Mac, and reload it after each change without a rebuild. MobAI runs flutter attach, flutter run or Metro itself, because an agent cannot press r in them; the agent starts the session, edits code, reads the reload result and stops it over HTTP. Use when iterating on a Flutter or React Native app's UI or logic, when asked to "hot reload", "hot restart", "fast refresh", "run it with live reload", "see changes without rebuilding", "run flutter on my iPhone from Windows", or when a build, install and launch cycle is slowing work down.
---

# Hot reloading mobile apps

MobAI runs the hot reload tool and keeps it running; you drive it with four
calls. Saving a file reloads the app on its own (Flutter's file watcher,
React Native's Fast Refresh), so most of the loop is: edit, save, read the
result, check the screen.

Never run `flutter run`, `flutter attach` or Metro yourself to get hot
reload: they wait for a key press you cannot send.

```sh
API="$MOBAI_API_URL/api/v1"     # MobAI's local API (http://127.0.0.1:8686/api/v1)
APP="$API/apps/$MOBAI_APP_ID"   # this app's App Builder routes
```

## Step 1: claim the device

```sh
curl -s -X POST "$API/devices/claim" -d "{\"device\":\"<device id>\",\"owner\":\"$MOBAI_CLAIM_OWNER\"}"
# answers {"deviceId": ..., "leaseToken": ...}; send the token on every device call
```

## Step 2: a debug build on the device, when the session needs one

| Framework and device | What the session runs | Needs an installed debug build |
|---|---|---|
| Flutter, iPhone, machine without Xcode | `flutter attach` through MobAI's iOS device | yes, plus its bundle id |
| Flutter, Android, or iOS on a Mac with Xcode | `flutter run`, which builds and installs | no |
| React Native or Expo, iPhone or simulator | Metro, then the app launched pointed at it | yes, plus its bundle id |
| React Native or Expo, Android | not this route: run `npx expo run:android` or `npx react-native run-android` in the background; they start Metro and Fast Refresh | no |
| Flutter or React Native, CI simulator | the session on the simulator's GitHub runner, attached to the build the job installed; each reload first sends the folder as it is then | no |

Without Xcode, iOS builds run on GitHub. Only a signed build installs on an
iPhone: make a development build (the `setting-up-mobile-apps` skill sets up
`signing/ios` with `development` if `GET $APP` does not list it in
`ios.kinds`). For a simulator on this machine the kind is `simulator`. The
`ci-simulator` kind is not a build: it starts a simulator on GitHub with a
build of the folder as it is. With `"noRebuild": true` it starts on the last
app a CI simulator built for this repository instead, minutes sooner (it
builds when there is none); a hot reload session then brings your edits in.
Changed native dependencies need the build, so leave `noRebuild` out then.

```sh
curl -s -X POST "$APP/builds" -d '{"platform":"ios","kind":"ci-simulator","noRebuild":true}'
```

```sh
curl -s -X POST "$APP/builds" -d '{"platform":"ios","kind":"development"}'   # or "simulator"
curl -s "$APP/builds/active"                    # poll until phase is done or failed
curl -s "$APP/builds"                           # the finished build's id
curl -s -X POST "$APP/builds/<build id>/install" -H "X-Lease-Token: <leaseToken>" -d '{"device_id":"<device id>"}'
```

App Builder's GitHub builds are debug builds unless `mobai.json` sets
`ios.configuration` to `Release`; store builds are release builds, and hot
reload needs debug. The bundle id is the iOS target's `bundleId` in `GET $APP`.

## Step 3: start the session

```sh
curl -s -X POST "$API/devices/<device id>/dev" -H "X-Lease-Token: <leaseToken>" \
  -d '{"framework":"flutter","dir":"<absolute path of the project folder>","bundleId":"<bundle id>"}'
```

- `framework`: `flutter` or `react-native` (Expo too).
- `dir`: the project's folder, the one with `pubspec.yaml` or `package.json`.
- `bundleId`: when step 2 says the session needs it.
- `metroPort`: React Native only, 8081 when left out. A Metro already
  listening there is used as it is.

The call answers when the app runs with hot reload on, which takes seconds
for `flutter attach` and React Native, and as long as a build for
`flutter run`. 200 is running; 422 carries the failed session with `error`
and the `log` that says why. `flutter attach` hot restarts the app once as it
starts, so it runs the code in the folder: that is the first `Restarted
application` in the log. A project that pins Flutter with fvm runs on its
pinned version, whatever `flutter` on PATH is.

```json
{"deviceId": "...", "framework": "flutter", "state": "running",
 "vmServiceUrl": "http://127.0.0.1:50123/kKx2tUbXYuM=/", "log": ["..."]}
```

## Step 4: change code and check

1. Edit and save. Flutter reloads on saves of `.dart` files under `lib/`;
   React Native's Fast Refresh reloads on save. On a CI simulator saving
   reloads nothing: when your edits are ready, call the reload below. It
   sends the folder as it is to the runner and reloads, in a few seconds.
2. Read what happened:

   ```sh
   curl -s "$API/devices/<device id>/dev?lines=40"
   ```

   Flutter prints `Reloaded N of M libraries`; a compile error prints the
   error and `Try again after fixing the above error(s)`. App output is
   prefixed `[app]`, Metro's `[metro]`.
3. Check the screen with MobAI (the `controlling-mobile-devices` skill).

Reload by hand, or hot restart when a hot reload cannot show the change
(Flutter: `main()`, `initState`, globals and static fields, enum or class
shape changes):

```sh
curl -s -X POST "$API/devices/<device id>/dev/reload" -d '{"restart":true}'
# answers the session with "result": what Flutter printed; 422 when the code does not compile
```

For React Native a reload asks Metro to reload the app.

## When hot reload is not enough

Native code (Swift, Kotlin, Objective-C, Java), native dependencies,
`Info.plist`, entitlements, app icons and `pubspec.yaml` plugins need a new
build: stop the session, rebuild and install (step 2), start again.

On a CI simulator, rebuild on its runner instead. This builds the folder as
it is with the job's own build, installs and launches it, and answers when
done (minutes; 422 with the end of the build's log when it fails). It ends a
running session. Changed dependencies still need a new CI simulator.
A CI simulator closes 10 minutes after it joins unless it is in use, and past
that 5 minutes after its last use. Stopping its bridge (`stop_bridge`, or
`POST $API/devices/<device id>/bridge/stop`) ends it and its GitHub run.

```sh
curl -s -X POST "$API/devices/<device id>/rebuild" -H "X-Lease-Token: <leaseToken>" \
  -d '{"dir":"<absolute path of the project folder>"}'
```

## Step 5: stop

```sh
curl -s -X DELETE "$API/devices/<device id>/dev"
```

Stopping ends the tool MobAI started and the app. A Metro that was already
running stays. Starting a new session on the device stops the old one.

## Problems

| What the session says | What to do |
|---|---|
| `the developer is not trusted` | Ask the user to open Settings > General > VPN & Device Management on the iPhone and trust the developer, then start again. |
| no Dart VM service URL | The installed build is a release build: make a debug one. |
| React Native app shows a red screen about the bundle | The iPhone and this computer must be on the same Wi-Fi. The first launch asks for Local Network access: ask the user to allow it, then start again. |
| `flutter` or `npx` not found | Flutter or Node.js is not installed on this machine; tell the user what to install. |
| `this CI device cannot rebuild or hot reload` | The CI simulator was started by an older mobai-ci. Start a new one. |

Only one Flutter session through MobAI's iOS device runs at a time: Flutter
keeps one `mobai-ios` device per user, and starting a session rewrites it.
