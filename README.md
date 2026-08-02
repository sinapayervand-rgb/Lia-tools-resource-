# Lia Tools — v1 (ZIP to APK)

Lia Tools is an Android app with one job in this first version: turn a ZIP of an
HTML/CSS/JS web project into a real, installable, signed APK — entirely on-device,
offline, with a clean dark-themed 4-step wizard.

This delivery contains **two Android Studio projects**:

```
LiaTools/                  <- the actual product. Open this in Android Studio.
LiaTools-ShellTemplate/    <- a tiny companion "shell" app. Build it once (see below),
                               then drop its output APK into LiaTools/ as an asset.
docs/BUILD_AND_ARCHITECTURE.md   <- how it all fits together, in depth
```

## Why two projects?

Android has no on-device compiler — `aapt2`, `d8`, and `apksigner` are desktop
JDK/SDK tools. So "build an APK on a phone" can't mean "run a real Android build
on a phone." What it *can* mean, and what every real offline APK-builder app
does, is: ship a **pre-built shell app** (a WebView wrapper) and, at runtime,
byte-patch a fresh copy of it — swap in the user's web files, swap the icon,
rewrite the app name/package/version inside the compiled manifest, and re-sign
it. That's exactly what `LiaTools/core/ziptoapk/ApkBuilder.kt` does.

`LiaTools-ShellTemplate` is that shell. It's a two-screen-simple WebView app.
You (the developer) build it **once** with a normal Gradle/Android Studio build,
then ship the resulting APK inside `LiaTools`'s assets. From then on, Lia Tools
never needs a compiler — it just re-skins that one prebuilt APK per build.

## Quick start

1. Open `LiaTools-ShellTemplate/` in Android Studio → Build → Build APK(s).
2. Copy the output APK to:
   `LiaTools/app/src/main/assets/template/shell_template.apk`
3. Open `LiaTools/` in Android Studio → Run.
4. On the device: Lia Tools → 📦 ZIP to APK → walk the 4-step wizard.

No network access is required at any point after step 1 (Gradle dependency
download aside) — the whole ZIP→APK pipeline runs on-device with plain Java/
Kotlin (ZIP I/O, a hand-written binary-XML manifest patcher, and a hand-written
PKCS#7 signer). See `docs/BUILD_AND_ARCHITECTURE.md` for exactly how each piece
works and what's intentionally simplified for v1.

## What's implemented (not a mockup)

- **Step 1–3 UI**: real file pickers (SAF) for the ZIP and icon, live validation
  (package-name format, required fields), Compose animations between steps.
- **ZIP extraction**: `SourceZipInspector` — streams the ZIP, guards against
  zip-slip path traversal, validates it's non-empty.
- **Icon processing**: `IconProcessor` — decodes any PNG/JPG, center-crops to
  square, renders all 5 launcher-icon densities.
- **Manifest patching**: `AxmlManifestPatcher` — parses the compiled binary
  AndroidManifest.xml inside the shell template and rewrites `package`,
  `android:versionName`, `android:versionCode`, and the application label,
  without needing aapt2.
- **APK assembly**: `ApkBuilder` — copies the shell template's ZIP entries
  through, substituting the manifest, the launcher icons, and injecting the
  user's web project under `assets/www/`.
- **Signing**: `ApkSigner` + `Pkcs7SignedDataBuilder` — a from-scratch APK
  Signature Scheme v1 (JAR signing) implementation: MANIFEST.MF, CERT.SF,
  CERT.RSA (hand-rolled DER/PKCS#7), signed with a bundled self-signed debug
  keystore (`assets/keystore/lia_debug.keystore`, generated via `keytool`).
- **Output**: install directly (`ACTION_VIEW` on the APK) or save to
  `Downloads/LiaTools/` via `MediaStore`.

## Known v1 simplifications (documented, not hidden)

- Output isn't `zipalign`ed — affects resource-mmap performance only, not
  installability.
- Only APK Signature Scheme v1 is produced (no v2/v3 block) — accepted by
  every Android version, but a `targetSdk` that enforces v2-only min signing
  would reject it. This shell template deliberately targets a config where v1
  is sufficient.
- The signing key is a shared, bundled debug key — fine for personal/offline
  installs, **not** for Play Store distribution.
- The application label can only be patched when the shell template declares
  `android:label` as a literal string (it does, by design) rather than
  `@string/app_name` — see the big comment at the top of the shell template's
  `AndroidManifest.xml`.

## Architecture for future tools

`core/tools/ToolRegistry.kt` is the seam: Home renders whatever's registered
there, and nothing about `HomeScreen` needs to change to add a second tool.
Each future tool gets its own `core/<tool>/` (logic) and
`ui/screens/<tool>/` (UI) package, following the same shape as `ziptoapk/`.
