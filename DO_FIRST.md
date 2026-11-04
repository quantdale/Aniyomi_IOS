# AniYomi iOS — DO FIRST

This is the pre-implementation feasibility checklist.

Do **not** begin building the production iPhone application until the required gates below have been tested. Do not modify the upstream reference repositories:

```text
aniyomi/
aniyomi-extensions/
extensions-source/
```

The production target repository is currently research-only. The exploration report is:

```text
Aniyomi_IOS/docs/ANIYOMI_IOS_EXPLORATION.md
```

The first experiments must be disposable and must live outside the production target. Do not create the real SwiftUI application yet.

## What must be obtained first

You need temporary access to a Mac that can connect to the physical iPhone 16 Pro by USB.

Preferred options:

1. Borrow a MacBook or Mac mini.
2. Use a family member’s or friend’s Mac.
3. Use a USB-capable remote Mac.
4. Use a cloud Mac only for compilation; it usually cannot replace a physical USB/device test.

The Mac should have:

- macOS compatible with the Xcode version selected for the phone;
- a current stable Xcode release with device support for the phone’s exact iOS build;
- at least 30–50 GB free storage;
- a USB cable;
- access to the Apple Account used for personal signing.

Apple’s current Xcode table is the authority for matching macOS, Xcode, SDK, and device support:

<https://developer.apple.com/xcode/system-requirements/>

## Phase 1 — Windows preparation

### 1. Record the exact iPhone software version

On the iPhone:

1. Open **Settings**.
2. Open **General → About**.
3. Record:
   - Model name.
   - iOS version.
   - Build number.
   - Available storage.

Do not rely on the previously reported `iOS 26.5.5` until the phone confirms it.

### 2. Preserve repository boundaries

From PowerShell:

```powershell
Set-Location 'D:\Documents\tryPython\Aniyomi_project'

git -C .\aniyomi status --short
git -C .\aniyomi-extensions status --short
git -C .\extensions-source status --short
git -C .\Aniyomi_IOS status --short

openspec list --json
```

Expected result:

- upstream repositories are clean;
- `Aniyomi_IOS` contains only the exploration documentation;
- OpenSpec reports no active change.

Do not commit research backups, downloaded extension artifacts, credentials, cookies, or personal media.

### 3. Prepare optional migration samples

If you already use Tachimanga or Aniyomi:

1. Create an official backup from the application.
2. Preserve the original file extension and filename.
3. Store it outside Git.
4. Do not upload it publicly.
5. Do not bypass encryption or access controls.

Useful samples are:

```text
Tachimanga native backup
Tachimanga Tachiyomi-compatible backup
Aniyomi .tachibk backup
```

If no library exists yet, skip migration testing.

### 4. Do not start production implementation

Do not create:

- SwiftUI screens;
- production database models;
- real source adapters;
- production downloader code;
- production extension installers;
- a formal OpenSpec Proposal.

## Phase 2 — Prepare the Mac

### 5. Install and verify Xcode

On the Mac:

1. Install Xcode from the Mac App Store.
2. Open it once.
3. Accept the license.
4. Allow required components to install.
5. Run:

```bash
xcodebuild -version
sw_vers
xcode-select -p
```

Save this output with the research results.

### 6. Connect and trust the iPhone

1. Connect the iPhone by USB.
2. Tap **Trust** on the iPhone if asked.
3. Enter the device passcode.
4. In Xcode, confirm the iPhone appears as a run destination.
5. Enable **Developer Mode** on the iPhone:

```text
Settings → Privacy & Security → Developer Mode
```

6. Restart the iPhone if requested.

### 7. Add the Apple Account to Xcode

1. Open **Xcode → Settings → Accounts**.
2. Add the Apple Account.
3. Confirm Xcode shows a **Personal Team** if the account is not paid.

A free Personal Team is sufficient for initial testing, but Apple currently limits it to 10 App IDs, 3 devices, 3 installed apps per device, and seven-day provisioning. Apps must be rebuilt and reinstalled after expiry:

<https://developer.apple.com/help/account/basics/about-your-developer-account>

## Phase 3 — Create a disposable feasibility project

Create a separate temporary project outside `Aniyomi_IOS`, for example:

```text
~/AniyomiIOS-Feasibility/
```

This project tests only:

1. Extension runtime loading.
2. Offline media downloads.
3. AVPlayer playback.
4. Persistence and progress.
5. Backup and restore.
6. Device signing.

It must not become the production application.

Clone the repositories into a separate research directory if needed:

```bash
mkdir -p ~/AniyomiIOS-Research
cd ~/AniyomiIOS-Research

git clone <private-Aniyomi_IOS-repository-url> Aniyomi_IOS
git clone https://github.com/aniyomiorg/aniyomi.git aniyomi
git clone https://github.com/aniyomiorg/aniyomi-extensions.git aniyomi-extensions
git clone https://github.com/Kohi-den/extensions-source.git extensions-source
```

Do not edit these clones for the experiment.

## Phase 4 — Extension compatibility gate

This is the highest-priority gate.

### Inputs

Use:

- one current MangaDex manga extension artifact;
- one current Aniyomi anime extension artifact;
- the matching extension API/library version for each artifact.

The Windows investigation already found that a current MangaDex JAR is not self-contained. It requires host-provided Tachiyomi APIs, Android classes, Kotlin, OkHttp, Jsoup, serialization, Rx/Injekt, and related dependencies.

### Required test sequence

The disposable host must verify:

1. Repository signature and artifact hash are checked.
2. The JAR/class files can be loaded.
3. The extension entry point can be instantiated.
4. The source API can be queried.
5. Search returns source models.
6. Title details return metadata.
7. Episodes or chapters enumerate.
8. HTTP requests execute.
9. Headers and cookies are preserved.
10. HTML/JSON parsing works.
11. Video or page resolution returns usable models.
12. The extension does not require Android UI to perform source work.
13. A timeout or exception is isolated to that source.
14. The same host contract can be used by a native adapter fallback.

### Runtime requirement

Do not treat ordinary desktop HotSpot execution as proof of iPhone feasibility. The iPhone experiment must use an interpreter/no-JIT-compatible runtime or an equivalent safe execution mechanism.

The test must record:

- runtime used;
- whether JAR bytecode executes as data;
- missing classes;
- Android APIs that must be shimmed;
- Kotlin/library compatibility failures;
- HTTP/DOM failures;
- source operations that cannot be represented.

### Decision rule

If both manga and anime artifacts pass the real source operations, compatibility Level 4.5 remains a possible experimental direction.

If the runtime fails, use:

```text
Aniyomi metadata/source IDs
        ↓
portable conversion where possible
        ↓
native Swift adapters for required sources
```

Do not promise unchanged APK/JAR execution unless this device test passes.

## Phase 5 — Offline anime download gate

Use authorized/public test media or locally controlled fixtures. Do not build DRM bypasses, CAPTCHA bypasses, or access-control circumvention.

### Direct MP4 test

1. Resolve a direct MP4 URL.
2. Include required `User-Agent`, `Referer`, and cookie behavior.
3. Start a background download.
4. Close the disposable app.
5. Reopen it.
6. Confirm download state persisted.
7. Interrupt Wi-Fi.
8. Restore Wi-Fi.
9. Confirm retry/resume behavior.
10. Verify the completed file.
11. Atomically move it from temporary storage to persistent storage.
12. Play it with `AVPlayer`.

### HLS test

1. Resolve an HLS master playlist.
2. Use `AVAssetDownloadURLSession` or the selected materializer.
3. Test required headers/cookies.
4. Close and reopen the app.
5. Enable Airplane Mode.
6. Confirm offline playback.
7. Confirm subtitle tracks if present.
8. Record whether custom materialization is needed.

Apple’s offline playback documentation is the authority for supported HLS behavior:

<https://developer.apple.com/documentation/avfoundation/offline-playback-and-storage>

### Expiring URL test

1. Use a short-lived signed URL.
2. Allow it to expire.
3. Attempt the download.
4. Confirm the record becomes `needsResolution`.
5. Resolve a fresh URL.
6. Retry safely.
7. Confirm no corrupted final file remains.

### Required download evidence

Record pass/fail for:

```text
direct MP4
HLS
custom headers
cookies
pause/resume
process termination
network interruption
URL expiry
Airplane Mode playback
storage verification
```

Do not claim background downloads are guaranteed. iOS controls background scheduling and may defer or terminate work.

## Phase 6 — Player gate

The disposable player must test:

- local MP4 playback;
- local HLS/offline asset playback;
- playback speed of at least 2.5×;
- exact playback-position persistence;
- seek gestures;
- brightness gesture;
- volume gesture;
- orientation lock;
- screen/control lock;
- automatic next episode;
- Picture in Picture;
- embedded subtitle tracks;
- unsupported-file handling;
- Open In/share fallback.

Record which formats play natively and which require remuxing or external-player fallback.

## Phase 7 — Persistence and backup gate

Create synthetic records for:

```text
source
title
episode/chapter
library state
watch/read progress
history
download state
extension state
preferences
```

Verify:

1. Same-looking titles from different sources remain separate.
2. Identity uses source plus source-local key/URL.
3. Anime seconds survive relaunch.
4. Manga page progress survives relaunch.
5. Download state survives termination.
6. Partial files can be recovered or discarded safely.
7. Orphaned media can be detected.
8. A metadata backup can be created.
9. A backup can be restored transactionally.
10. Corrupt backups do not partially overwrite data.
11. Normal backups do not require embedding media.

For migration samples:

1. Inspect legitimate Tachimanga exports first.
2. Inspect Aniyomi `.tachibk` exports if available.
3. Map source IDs and source-local URLs.
4. Map library/category state.
5. Map anime seconds and manga page progress.
6. Record fields that cannot be mapped.

Do not reverse-engineer protected information or bypass backup encryption.

## Phase 8 — Signing and installation gate

Use direct Xcode installation first:

1. Select the physical iPhone as the run destination.
2. Select the Personal Team under **Signing & Capabilities**.
3. Use one application identifier.
4. Build and run the disposable feasibility project.
5. Disconnect the Mac.
6. Launch the app on the iPhone.
7. Record installation, launch, and expiry behavior.

Do not add App Groups, extensions, or unnecessary capabilities during this test.

## Phase 9 — Optional Windows-assisted SideStore path

Only evaluate SideStore after direct Xcode installation works.

Current SideStore prerequisites include:

- iPhone/iPad/iPod touch running iOS 15 or newer;
- 64-bit Windows 8 or newer, macOS, Linux, or supported Chromebook;
- Apple Account;
- Wi-Fi;
- LocalDevVPN;
- iTunes or Apple device support;
- `iloader` for the initial installation.

Official instructions:

- <https://docs.sidestore.io/docs/installation/prerequisites>
- <https://docs.sidestore.io/docs/installation/install>

The general Windows flow is:

1. Install the required iTunes/Apple device support.
2. Install `iloader`.
3. Connect the iPhone by USB.
4. Trust the computer.
5. Open `iloader`.
6. Sign in with the Apple Account.
7. Select the iPhone.
8. Install SideStore Stable.
9. Trust the developer profile on the iPhone.
10. Enable Developer Mode.
11. Install and connect LocalDevVPN.
12. Open SideStore.
13. Sign in with the same Apple Account.
14. Refresh the seven-day signing status regularly.

SideStore helps with periodic IPA refresh. It does not replace Xcode, the iOS SDK, the physical-device runtime tests, or the extension compatibility gate.

## Phase 10 — Evidence to record

Create a private research log containing:

```text
Mac model
macOS version
Xcode version
iPhone model
iOS version/build
Apple signing result
runtime name/version
extension artifact names/versions/hashes
missing classes
source API results
HTTP/DOM results
direct download result
HLS download result
expiry recovery result
AVPlayer result
SwiftData/relaunch result
backup/restore result
Tachimanga migration result
SideStore result
```

Do not include passwords, tokens, cookies, private backup contents, or confidential URLs in exported logs.

## Final decision rules

### If the JVM/no-JIT gate passes

The future Proposal may include an experimental compatibility runtime, but it must still provide native/script fallbacks and explicit trust boundaries.

### If the JVM/no-JIT gate fails

The future Proposal should use source metadata compatibility, portable conversion where feasible, and native Swift adapters for the required initial sources.

### If HLS/offline playback fails

The future Proposal must specify direct-file support first and document HLS limitations or a separate materialization strategy. Streaming must not silently replace offline downloads.

### If no Mac is available

The Proposal can be written with these as unresolved critical gates, but that is not recommended. A short USB-capable Mac session is the highest-value next action.

## Definition of done before Proposal

The pre-Proposal phase is ready when:

- exact iPhone/Xcode compatibility is confirmed;
- a disposable project installs on the physical iPhone;
- current manga and anime extension artifacts have a documented runtime result;
- direct MP4 and HLS behavior are documented;
- expiring URL recovery is documented;
- local playback and progress persistence are documented;
- backup/migration limitations are documented;
- free signing and installation limits are documented;
- all failures have logs and reproducible fixtures;
- the production repository still contains no application implementation.

At that point, create the formal OpenSpec Proposal.
