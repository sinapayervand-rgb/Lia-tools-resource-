# Lia Tools — Build & Architecture

## Why you get a source project, not an .apk

This was built in a sandbox with no Android SDK (no `aapt2`, no `d8`, no
`apksigner`, no emulator) — only a JDK and Gradle config files. That's exactly
why the ZIP→APK *pipeline itself* was written to work without those tools at
runtime (see below) — but it also means I can't compile *this* project into an
.apk here. Open `LiaTools/` in Android Studio (Arctic Fox or newer, AGP 8.5,
Kotlin 1.9) and hit Run — it's a normal Gradle project, nothing exotic required
to build it.

## The two projects, and why

```
LiaTools/                 the product itself
LiaTools-ShellTemplate/   a tiny WebView app, built once, consumed as an asset
```

Android has no on-device compiler. A phone can't run `aapt2`/`d8` the way a
desktop build does. So "build an APK on a phone" really means: ship a
pre-compiled shell app, and at runtime *re-skin* a fresh copy of it — swap its
web assets, its icon, and a few identity fields in its already-compiled
manifest — then re-sign it. That's what every offline "website to APK" tool
does, and it's what `ApkBuilder` implements.

Build the shell once:
1. Open `LiaTools-ShellTemplate/` → Build APK(s).
2. Copy the output to `LiaTools/app/src/main/assets/template/shell_template.apk`.

From then on `LiaTools` needs no compiler at all — everything in Step 4 is
plain ZIP/byte manipulation.

## The ZIP → APK pipeline, step by step

1. **`SourceZipInspector`** streams the user's ZIP via `ZipInputStream`,
   guards against zip-slip (`../../` path traversal), rejects empty archives.
2. **`IconProcessor`** decodes the chosen PNG/JPG, center-crops to square,
   renders it at all 5 mipmap densities (48–192px).
3. **`AxmlManifestPatcher`** — the interesting part. A compiled
   `AndroidManifest.xml` is a binary chunk format (`ResChunk` records); string
   attribute values are just indices into a string pool at the top of the
   file. Rather than parsing/rebuilding the whole tree, the patcher:
   - decodes the string pool,
   - linear-scans element/attribute chunks to find the byte *offsets* of the
     `package`, `android:versionName`, `android:versionCode`, and
     `android:label` attribute value fields,
   - **appends** the new string values to the pool (existing indices never
     move, so nothing else needs to change),
   - overwrites those 4-byte offset fields in place,
   - re-encodes just the string-pool chunk and splices it back in.
   Full chunk-format notes are in the file's own doc comment.
4. **`ApkBuilder`** copies the shell template's ZIP entries through unchanged,
   substituting: the patched manifest, the re-rendered icons (matched by a
   density regex against whatever aapt2 named the entry — not a hardcoded
   path), and the user's web files under `assets/www/`.
5. **`ApkSigner`** implements APK Signature Scheme v1 (JAR signing) from
   scratch: builds `MANIFEST.MF` (per-entry SHA-256 digests), `CERT.SF`
   (digest-of-digests), and `CERT.RSA` — a hand-encoded PKCS#7 `SignedData`
   block (see `Der.kt` / `Pkcs7SignedDataBuilder.kt`) — signed with the
   bundled self-signed debug key (`assets/keystore/lia_debug.keystore`,
   generated once via `keytool`, the same trust model Android Studio's own
   debug builds use).

Every step reports progress via a `Flow<BuildEvent>` that drives Step 4's
circular progress indicator and stage label.

## Known v1 limitations (intentional, documented)

- No zipalign — affects resource mmap performance only, not installability.
- v1 signing only, no v2/v3 signing block — accepted on every Android version
  as long as the app doesn't declare a `minSdkVersion` that requires v2.
- Shared debug signing key — fine for sideloaded/offline installs, not for
  Play Store.
- Manifest patching only rewrites *literal* string attributes — the shell
  template's `android:label` is deliberately written as a raw string (not
  `@string/app_name`) so this works. If you ever edit the shell template,
  keep that literal.

## Extending Lia Tools with a second tool

Add `core/<tool>/` for logic and `ui/screens/<tool>/` for UI, register a
`LiaTool` entry in `core/tools/ToolRegistry.kt`, add its route to
`ui/LiaNavGraph.kt`. `HomeScreen` needs zero changes — it just renders
whatever the registry exposes.
