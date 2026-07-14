# CallSync Android app — skeleton

The Android half of the pipeline: detects calls, checks them against Zoho
leads via your backend, records matched calls, and uploads everything back.
Open this folder in **Android Studio** (Hedgehog or newer) to build it — it
cannot be compiled or run in the environment this was written in, so unlike
the backend, this hasn't been test-executed. Read it carefully before
running.

## What's real here

- **Gradle project structure, manifest, and permissions** — genuinely
  correct and complete for a default-dialer app.
- **`CallSyncInCallService`** — the actual call-detection pipeline. Once
  this app holds the default dialer role, Android calls into this on every
  call state change. It really does call your backend's create → match →
  record → end → upload → sync sequence in order.
- **`CallSyncApi` / `ApiClient`** — a real Retrofit client matching your
  backend's endpoints field-for-field.
- **`ContactSync`** — really reads device contacts via `ContactsContract`
  and checks each against Zoho leads through your backend.
- **`MainActivity`** — really requests runtime permissions and really
  triggers Android's default-dialer role picker via `RoleManager`.

## What's stubbed, and why

- **Zoho OAuth login** — the button exists but doesn't open a real OAuth
  URL yet. You need your Zoho Client ID first (see backend's
  `HOSTINGER_DEPLOY.md` / main `README.md` for registering one), then wire
  the URL into `MainActivity.kt`'s `btnZohoLogin` click handler — the exact
  redirect flow is documented in a comment right there.
- **`DialerActivity`** — a placeholder screen. Android requires a default
  dialer app to handle `ACTION_DIAL`, but building a real dialpad + contact
  picker UI is a separate, sizeable task. Read the comment at the top of
  `DialerActivity.kt` — there's a pragmatic shortcut described there (keep
  the phone's existing dialer for actually placing calls; this app still
  gets call-state events as long as it holds the default dialer role).
- **Recording reliability across devices** — read the comment block at the
  top of `CallRecorder.kt` before testing on a real device. `VOICE_CALL`
  audio capture is gated by the phone manufacturer, not just by Android
  permissions — it's known to work on Samsung/Xiaomi/OnePlus-type devices
  and known to fail on Pixel/stock Android, regardless of default-dialer
  status. Test on your actual target device before assuming this half of
  the pipeline works.
- **Retry/offline handling** — if the network call to your backend fails
  mid-call (`onCallEnded`), the recording and log are currently just lost.
  A production build should queue failed uploads via `WorkManager` and
  retry instead.

## Getting an APK without installing Android Studio

If you just want an installable `.apk` and don't want to set up a local
Android dev environment, this project includes a GitHub Actions workflow
(`.github/workflows/build-apk.yml`) that builds one automatically:

1. Push this project to a new GitHub repository (private is fine):
   ```
   git init
   git add .
   git commit -m "Initial CallSync Android skeleton"
   git remote add origin https://github.com/yourusername/your-repo.git
   git push -u origin main
   ```
2. Go to the **Actions** tab on GitHub — a "Build debug APK" run should
   start automatically. If it doesn't, click **Run workflow** to trigger it
   manually.
3. Once it finishes (a few minutes), open the completed run and download
   the `callsync-debug-apk` artifact from the **Artifacts** section at the
   bottom of the page. Unzip it — `app-debug.apk` is inside.
4. Transfer that `.apk` to an Android phone and install it (you'll need to
   allow "install from unknown sources" for whichever app you use to open
   it, since this isn't distributed through the Play Store).

This build produces a **debug** APK — fine for testing on your own devices,
not signed for Play Store distribution. Release signing is a separate step
or if/when you're ready to distribute this more broadly.

Note: this workflow doesn't rely on a committed Gradle wrapper
(`gradlew`/`gradle-wrapper.jar`) — those binary files couldn't be generated
in the environment this project was written in, since creating them
requires downloading Gradle itself. The workflow provisions Gradle directly
on GitHub's runner instead. If you open this project in Android Studio
locally, it will generate the wrapper for you automatically the first time
you sync the project — you don't need to do anything manually for that.

## Before you build

1. Make sure your backend (from the previous zip) is running somewhere
   reachable from a phone or emulator — either your Hostinger deployment
   or `localhost` via the Android emulator's `10.0.2.2` alias (already set
   as the debug `API_BASE_URL` in `app/build.gradle.kts`).
2. For a release build, change the `release` build type's `API_BASE_URL`
   in `app/build.gradle.kts` to your real `https://` backend domain, and
   remove `android:usesCleartextTraffic="true"` from the manifest — that
   flag only exists so the emulator can reach plain `http://` during dev.
3. Add a real launcher icon — this skeleton doesn't include one
   (`android:icon` was left out of the manifest for that reason). Android
   Studio's **Image Asset Studio** (right-click `res` → New → Image Asset)
   generates one in a minute.

## Testing without a second phone to call

The Android emulator can simulate calls without needing a real SIM:
```
adb emu gsm call +15551234567
```
This triggers `onCallAdded` in `CallSyncInCallService` exactly as a real
call would, so you can verify the create/match/record pipeline end-to-end
against your running backend. (Recording itself may not produce real audio
in the emulator — verify that specifically on a physical device.)

## Suggested next step

Given the `DialerActivity` stub above, the single highest-leverage next
step is probably **not** building a full dialpad UI — it's confirming
`CallSyncInCallService` correctly receives call events while the rep's
existing phone dialer places the calls, with this app just holding the
default-dialer role in the background. That validates the entire pipeline
without needing to build real UI first.
