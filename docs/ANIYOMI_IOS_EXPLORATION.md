# Aniyomi iOS — OpenSpec Explore Report

Status: exploration only. No application source, formal proposal, or upstream change was created.

Date of inspection: 2026-08-08. Time-sensitive claims should be checked again when implementation begins.

Evidence labels:

- **Confirmed** — directly observed in checked-out source or authoritative platform documentation.
- **Strongly indicated** — supported by multiple paths or a public product with the same behavior, but one detail remains private.
- **Inferred** — architectural consequence of confirmed behavior.
- **Unknown** — requires a prototype, a legitimate exported file, or unavailable public information.

## A. Executive conclusion

The locked product is feasible as a personal, iPhone-only application. Local anime/manga data, source-oriented discovery, offline files, progress/history, a custom player/reader, manual refresh, backup, and a Windows-heavy workflow all have viable iOS implementations.

The highest-risk requirement is existing Aniyomi extension compatibility. An extension is executable Kotlin/JVM code inside an Android package or JVM artifact, not a declarative JSON record. It calls Tachiyomi/Aniyomi APIs, OkHttp, Jsoup, Android preferences/UI classes, and sometimes JavaScript, WASM, crypto, or nested video extractors. The current Aniyomi host uses Android PackageManager and Dalvik class loaders; those mechanisms do not exist on iOS.

The evidence does not justify declaring compatibility impossible:

1. Apple requires signed native executable code and prevents ordinary JIT/self-modifying native pages, but an interpreter may execute bytecode held as data.
2. OpenJDK has an official iOS port using the Zero C interpreter because iOS ARM devices cannot use HotSpot JIT generation: [OpenJDK iOS platform details](https://openjdk.org/projects/mobile/ios.html).
3. Tachimanga’s public Tachidesk fork downloads an extension APK, parses metadata, converts DEX to a JAR with dex2jar, and loads classes through URLClassLoader. Its fork also contains iOS-oriented networking/SQLite substitutions. The closed Tachimanga application/runtime that embeds this server is not public, so reproducing it in a new native Swift app is **unknown**, not confirmed.
4. Current Keiyoushi extension builds publish both signed APKs and signed JARs (the index field is Resources.jarUrl), making a JVM-input path more plausible than a DEX-only path.

Compatibility levels used here:

| Level | Meaning | Feasibility |
|---|---|---|
| 0 | Independent native sources | Easy |
| 1 | Aniyomi-like install/enable/update UX | Easy |
| 2 | Repository metadata, package IDs, source IDs, URLs | Easy and valuable |
| 3 | Source-level conversion or portable scripts | Feasible for a curated subset |
| 4 | Source-compatible API shims with recompilation | Feasible for controlled builds |
| 4.5 | Unchanged APK/JAR input transformed to JVM classes and executed by an embedded no-JIT JVM | Technically demonstrated by the Tachimanga/Suwayomi path; not yet reproducible here |
| 5 | Direct Android DEX/APK execution as an Android runtime | Not a sensible native iOS target |

The recommended direction for Proposal is a **runtime-pluggable native host**. It should normalize source identity, networking, preferences, media/page models, downloads, persistence, and diagnostics, while supporting:

1. a gated embedded-JVM compatibility runtime (first feasibility spike);
2. a constrained JavaScript/portable-script runtime for sources that can be converted safely; and
3. native Swift adapters for initial critical sources and exceptional extractors.

The Proposal should not promise Level 4.5 until a current manga extension and an anime extension load, call the real source API, perform HTTP/DOM work, and return normalized models on the target iPhone. If that spike fails, Level 3 plus native adapters is the sustainable fallback and the host should already support it.

Offline anime downloads remain the primary consumption path. Direct files and HLS need distinct materializers. Expiring URLs require a persisted “needs re-resolution” state because iOS cannot run extension code while a suspended app waits for a new signed URL.

Windows can host most domain/source/fixture development and Git/CI work. A Mac with Xcode remains unavoidable for final iOS compilation, simulator/device testing, signing, and first iPhone installation. A free Personal Team profile currently has 10 App IDs, 3 devices, 3 installed apps per device, and seven-day expiry; [Apple’s account guidance](https://developer.apple.com/help/account/basics/about-your-developer-account) documents those limits.

## B. Workspace and repository map

The inspected workspace is:

~~~text
D:\Documents\tryPython\Aniyomi_project\
├── aniyomi/             # upstream/reference Android application
├── aniyomi-extensions/  # upstream/reference legacy extension repository
├── extensions-source/   # Kohi-den current anime source repository
├── Aniyomi_IOS/         # target repository; originally README-only
├── openspec/            # root OpenSpec configuration; no active changes
└── .codex/skills/       # local skills, including openspec-explore
~~~

Repository evidence:

- aniyomi remote: https://github.com/aniyomiorg/aniyomi.git; inspected commit 1c2876e93e8ea07ef8e53856082aa442690c625a (2026-08-06). It remained unmodified.
- aniyomi-extensions remote: https://github.com/aniyomiorg/aniyomi-extensions.git; inspected as legacy extension/build reference.
- extensions-source remote: https://github.com/Kohi-den/extensions-source.git; representative files are src/en/av1encodes/.../Av1Encodes.kt and src/en/animekai/.../AnimeKai.kt.
- Aniyomi_IOS remote: https://github.com/quantdale/Aniyomi_IOS.git; clean and empty of app implementation before this report.
- openspec/config.yaml selects the spec-driven schema. openspec list --json reported no active changes.
- No root AGENTS.md was present. The current extension repository has its own AGENTS.md; it was read for research and not edited.

The Android repository is reference-only. Its data models and algorithms are evidence, not code to copy into an iOS target.

## C. Aniyomi architecture findings

### C.1 Modules

The upstream tree separates source API, metadata/core/archive utilities, SQLDelight data repositories, domain use cases, Android app services, and presentation. The useful conceptual boundary is:

~~~text
source implementation -> source API + HTTP/DOM/preferences
                    -> normalized titles/episodes/chapters/media/pages
host app            -> persistence/downloads/player/reader/history
~~~

The source API’s commonMain placement does not make it platform-neutral. It imports Android URI/context/network/preference types and the Gradle configuration currently targets Android for the relevant API.

### C.2 Source contracts and models

Confirmed paths:

- aniyomi/source-api/src/commonMain/kotlin/eu/kanade/tachiyomi/animesource/AnimeSource.kt
- aniyomi/source-api/src/commonMain/kotlin/eu/kanade/tachiyomi/animesource/AnimeCatalogueSource.kt
- aniyomi/source-api/src/commonMain/kotlin/eu/kanade/tachiyomi/animesource/online/AnimeHttpSource.kt
- aniyomi/source-api/src/commonMain/kotlin/eu/kanade/tachiyomi/source/MangaSource.kt
- aniyomi/source-api/src/commonMain/kotlin/eu/kanade/tachiyomi/source/CatalogueSource.kt
- aniyomi/source-api/src/commonMain/kotlin/eu/kanade/tachiyomi/source/online/HttpSource.kt

Anime sources expose stable source ID/name/language/capabilities, popular/latest/search/update operations, title details, episodes, and a newer hoster-to-video flow. Manga sources expose equivalent browse/search/details/chapter/page operations. AnimeHttpSource owns OkHttp helpers, Jsoup conversion, default parsers, and a source-ID algorithm derived from name/language/version through MD5.

Models include:

- title: source-local URL, title, alternate metadata, poster, description, genres, status;
- episode/chapter: source-local URL, name, number, upload date, group/scanlator, watched/read fields;
- media: URL, quality/resolution, headers, bitrate, subtitle/audio tracks, internal resolver data;
- page: ordered URL/image descriptors and optional headers;
- filters: text, checkbox, select, sort, grouped, tri-state, header/separator.

Several models import android.net.Uri; these need a Foundation URL boundary in Swift.

### C.3 Android extension lifecycle

Confirmed paths:

- aniyomi/app/src/main/java/eu/kanade/tachiyomi/extension/anime/util/AnimeExtensionLoader.kt
- aniyomi/app/src/main/java/eu/kanade/tachiyomi/extension/manga/util/MangaExtensionLoader.kt
- corresponding manager/repository/installer packages.

The host fetches an index, compares package/version/library metadata, downloads an APK, checks feature metadata and signatures, installs or privately copies it, reads ApplicationInfo/manifest metadata, creates a child-first or normal Dalvik class loader, reflects the entry class/factory, and exposes source instances. Anime library versions 12–16 and manga versions 1.4–1.5 were accepted at the inspected Aniyomi commit. Current Mihon/Keiyoushi sources already use lib 1.6, so the ABI is versioned and evolving.

An iOS host cannot call PackageManager, install an APK as an app, use Dalvik loading, or assume Android Application/Context/preferences.

### C.4 Network subsystem

aniyomi/app/src/main/java/eu/kanade/tachiyomi/network/NetworkHelper.kt configures OkHttp, cookies, cache, timeouts, Brotli, optional DoH/logging, and Cloudflare browser-interaction behavior. AndroidCookieJar bridges OkHttp to Android WebView CookieManager. The Cloudflare interceptor opens a JavaScript-capable WebView and waits for a browser challenge cookie; this is an interactive user flow, not a generic anti-bot bypass.

The iOS equivalents are URLSession, HTTPCookieStorage, WKWebView, and WKHTTPCookieStore. Synchronization is explicit. Interceptors, DoH, redirects, and browser cookies require a compatibility bridge.

## D. Extension deep dive

### D.1 Artifact and dependencies

Legacy extensions are signed Android APKs containing AndroidManifest.xml, classes.dex, resources, assets, icon, version metadata, and a signing certificate. Current Keiyoushi builds also create a signed JAR from class files/manifest/resources/assets. The relevant build task is gradle/build-logic/src/main/kotlin/io/github/keiyoushi/gradle/tasks/CreateExtensionJarTask.kt; publication is .github/scripts/publish-repo.py, with jarUrl added in .github/scripts/index.proto.

Typical dependencies include Kotlin/JVM and coroutines, serialization, OkHttp/Okio, Jsoup, RxJava, Android Application/Context/SharedPreferences/Handler/Looper/Uri/logging, AndroidX preferences, QuickJS/Duktape/WASM helpers, and source-specific crypto/JSON/archive/extractor libraries.

### D.2 Installation, updates, trust

Repository metadata includes name, package, APK/JAR URL, language, version, library version, content flags, source IDs/names/language/base URLs. A repository key fingerprint is used for trust; manual package/version/signature trust is also supported. The iOS manager can preserve the same UX while treating packages as data/scripts/JARs rather than native extensions.

Recommended package record:

- domain, extension ID, source IDs, name/language;
- version and minimum host API;
- artifact URLs, hash, repository signature;
- declared capabilities/allowed domains;
- install/enable/disable state and selected runtime;
- last error/diagnostic metadata;
- source-local identity and migration aliases.

### D.3 Preferences and filters

Configurable sources use Android SharedPreferences and PreferenceScreen widgets. Preferences cover domains/mirrors, language, server, quality, and source-specific behavior. iOS should map simple preference descriptors to SwiftUI forms; arbitrary Android preference UI should be a compatibility warning/capability rather than a promise of pixel parity.

### D.4 Disposable Windows artifact gate (completed 2026-08-08)

A current Keiyoushi MangaDex release was downloaded from the repository index for a read-only compatibility check. The index reports package `eu.kanade.tachiyomi.extension.all.mangadex`, extension library `1.4`, version `1.4.211`, a repository signing key, an APK URL, and a separate JAR URL. The JAR is 222,740 bytes and contains 144 ZIP entries: 127 class files, Kotlin module metadata, resources, and `META-INF/CERT.SF`/`CERT.RSA`. `jarsigner -verify` reports the JAR as verified, while warning that its certificate is self-signed and not timestamped; repository key/hash verification must therefore remain part of trust rather than relying on the JVM certificate chain. The APK is a separate 91,822-byte artifact containing `classes.dex`, `AndroidManifest.xml`, resources, and localized assets. The APK and JAR have different SHA-256 hashes, as expected for different packaging forms.

The disposable Java `URLClassLoader` probe successfully ran on the Windows Android Studio JBR, but loading the JAR failed immediately with `NoClassDefFoundError` for `eu/kanade/tachiyomi/source/SourceFactory` and `ConfigurableSource`. Re-running the same probe with HotSpot's `-Xint` interpreter mode produced the identical failures. `jdeps --ignore-missing-deps` independently reports missing `eu.kanade.tachiyomi.source*`, `eu.kanade.tachiyomi.network`, Android framework (`android.*`), Kotlin/Kotlin-reflect, kotlinx.serialization, OkHttp, Jsoup, Rx, and Injekt packages. This confirms that the published JAR is not self-contained: it is an extension payload that requires a compatible host API/runtime. A Windows JVM can load the class format, including interpreter-only execution, but this is not evidence that the same class graph or Android behavior works on iOS.

The static source corpus also confirms materially different complexity. Representative files measured 178–939 lines; import/function scans found Android imports in AnimeKai, MegaUp, MangaDex, and the AV1Encodes adapter, plus OkHttp/Jsoup/serialization and helper/extractor calls. Anime source inspection found direct file paths and HLS (`.m3u8`), VTT subtitle tracks, `User-Agent`/`Referer` headers, and source-specific quality preferences (`1080p`, `720p`, `480p`, `360p`). A disposable ranking fixture correctly normalized `2160p`, `1080p HEVC`, `FHD`, `1920 x 1080`, `720p`, `HD`, `480p`, and `360p` into numeric heights; unknown labels remain unresolved and require a source-label fallback.

The machine-level toolchain gate is also concrete: Git, Python, Node, .NET, and Android Studio JBR 21 are available; `swift`, `xcodebuild`, `adb`, standalone `java`, standalone `javac`, and Gradle on PATH are absent. The Gradle wrapper download/build was not treated as a completed build after exceeding the bounded Windows check. No Mac, Xcode SDK, iPhone, or signing validation is available in this workspace, so the no-JIT iOS JVM gate, AVPlayer/HLS offline gate, free-Personal-Team provisioning, and real-device background behavior remain unresolved external gates.

## E. Representative extension traces

### E.1 Anime: AV1Encodes

Files:

- extensions-source/src/en/av1encodes/src/eu/kanade/tachiyomi/animeextension/en/av1encodes/Av1Encodes.kt
- extensions-source/src/en/av1encodes/src/eu/kanade/tachiyomi/animeextension/en/av1encodes/Av1EncodesExtractors.kt
- extensions-source/src/en/av1encodes/src/eu/kanade/tachiyomi/animeextension/en/av1encodes/Av1EncodesModels.kt

Flow:

1. Search builds /search?q=...&page=... requests and parses cards with CSS selectors.
2. Details parse title/poster/description/genres/status.
3. Season/episode pages are fetched; anchors become SEpisode objects. Preferences select 1080/720/480/360 and link/torrent behavior.
4. The episode page contains inline JavaScript with an X-DDL-Token; a later request uses token/referer/origin/XHR headers.
5. JSON is decoded into direct watch/download/torrent variants and Video objects with quality/header/track data.
6. The Android host’s EpisodeLoader/HosterLoader chooses a candidate; the downloader or player consumes the normalized Video.

This is mostly HTTP, JSON, DOM, and preferences, so a source shim is plausible, but Kotlin/OkHttp/Android API compatibility is still required.

### E.2 Anime: AnimeKai/MegaUp

Files:

- extensions-source/src/en/animekai/src/eu/kanade/tachiyomi/animeextension/en/animekai/AnimeKai.kt
- extensions-source/src/en/animekai/src/eu/kanade/tachiyomi/animeextension/en/animekai/MegaUp.kt
- extensions-source/src/en/animekai/src/eu/kanade/tachiyomi/animeextension/en/animekai/MegaUpInterceptorOld.kt

Flow:

1. Search/latest/popular use HTML and filters.
2. Details parse title, poster, description, genres, status, studios, and producers.
3. Episode enumeration extracts an ID, calls a decode endpoint for a token, requests AJAX episode HTML, and stores an episode token in the episode URL.
4. Video enumeration repeats token decoding, requests server groups, decodes/decrypts links, and iterates nested hosters.
5. MegaUp calls a media endpoint, decodes JSON, locates an HLS master playlist or direct file, parses resolution/codecs, validates variants, and returns Video objects with User-Agent/Referer and VTT tracks.
6. Android side effects include Application, SharedPreferences, logging, Toast, preference widgets, and Handler/Looper.

This demonstrates that media resolution is a source/extractor graph with signed URLs, headers, cookies, external JSON/decryption services, HLS, and subtitle metadata—not a single core streaming call.

### E.3 Manga: MangaDex

Files in the current Keiyoushi checkout:

- src/all/mangadex/src/eu/kanade/tachiyomi/extension/all/mangadex/MangaDex.kt
- src/all/mangadex/src/eu/kanade/tachiyomi/extension/all/mangadex/MangaDexFilters.kt
- src/all/mangadex/src/eu/kanade/tachiyomi/extension/all/mangadex/dto/AtHomeDto.kt

Flow:

1. JSON search/popular/latest maps title, alternate titles, description, tags, status, and cover.
2. A paginated feed maps chapter URL/number/name/language/group/date.
3. /at-home/server/{chapterId} returns base URL, chapter hash, and ordered data/dataSaver filenames.
4. Final image URLs are derived from the current host/hash; helper code can re-resolve when the host/token expires.
5. Filters use checkbox/group/sort/select/tri-state descriptors and preferences include data-saver choice.

The stable identity is source ID plus title/chapter key, not the temporary image URL.

### E.4 Ownership table

| Operation | Extension | Shared library | Host/platform |
|---|---|---|---|
| Search/detail/episode/chapter parse | Yes | Default HTTP/DOM helpers | Normalize model |
| Hoster/video extraction | Yes; often nested | Some common extractors | Orchestrate/resolver |
| Direct/HLS retrieval | Usually descriptor | FFmpeg/extractor helpers | Download/materialize |
| Page retrieval | Usually descriptor | Image/archive helpers | Download/filesystem |
| Progress/history/library | No | No | Database/domain |
| Install/update/trust | No | Repository DTOs | iOS source manager |

## F. iOS compatibility matrix

| Capability | Android/JVM dependency | Native iOS equivalent | Difficulty | Finding |
|---|---|---|---|---|
| APK manifest | PackageManager | ZIP/manifest parser | Low | Metadata is preservable |
| DEX loading | Dalvik PathClassLoader | None | Critical | Needs JVM or conversion |
| JAR loading | JVM URLClassLoader | Embedded no-JIT JVM | Critical/high | Technically possible, unproven here |
| Source interfaces | Kotlin API classes | Swift protocols/bridge | Medium | Models map after Android removal |
| Coroutines/Flow | Kotlin runtime | async/await/actors | High for binary | Source conversion is non-mechanical |
| OkHttp | JVM/Android | URLSession wrapper | Medium/high | Cookies/interceptors need bridge |
| Jsoup | JVM DOM | SwiftSoup/DOM bridge | Medium | Selector parity needs fixtures |
| SharedPreferences | Android | UserDefaults/SwiftData | Medium | Descriptor mapping works |
| Context/Application | Android framework | Host capability object | High | Restrict native access |
| Handler/Looper | Android main scheduler | MainActor/tasks | Medium | Common cases bridge |
| Uri | Android | Foundation URL | Low/medium | Normalize at boundary |
| RxJava | JVM | AsyncSequence/Combine | Medium/high | Runtime shim or source rewrite |
| QuickJS/WASM | Android/native helpers | JavaScriptCore/Wasm/native adapter | High | Source-specific |
| WebView cookies | Android CookieManager | WKWebView cookie store | High | Explicit synchronization |
| HLS | OkHttp/extractors | AVAssetDownload/URLSession | Medium/high | Header/token spike needed |
| FFmpeg arguments | FFmpegKit | AVFoundation/media library | High | Do not assume MKV support |
| Package signature | APK cert | Repo signatures/hashes | Medium | Data trust, not app install |
| Failure isolation | Partial class-loader isolation | Actor/timeout; no cheap process | High | Treat runtime as untrusted |

## G. Compatibility strategy comparison

### A — Direct APK/DEX

An iOS app cannot install an Android APK or execute its DEX through an Apple loader. Apple validates executable code signatures and executable pages; [Apple Platform Security](https://support.apple.com/en-ie/guide/security/sec7c917bf14/web) documents the constraint. A DEX interpreter is theoretically possible but becomes a complete custom Android/JVM runtime and is not a sensible first implementation.

### B — Build-time conversion

An APK can be downloaded on Windows and converted to JAR, as Tachimanga’s public fork does, but a JAR still needs a JVM or AOT conversion. J2ObjC translates Java source and is not an Android-binary-to-Swift compiler; [J2ObjC](https://developers.google.com/j2objc) documents the distinction. A generic converter must understand Kotlin metadata, coroutines, generated serialization, reflection, Android classes, and bytecode-only libraries.

### C — Translator/compiler

Source-level Kotlin-to-script/Swift conversion can cover HTTP/DOM/JSON source code. It becomes brittle around reflection, static initialization, coroutine/Rx chains, Android preference UI, generated serializers, crypto/obfuscation, QuickJS/WASM, and custom WebView behavior. It should emit an unsupported-construct report rather than promise ecosystem-wide coverage.

### D — Compatibility runtime

Reimplementing source models, HTTP, DOM, cookies, preferences, filters, and hoster types is realistic. Executing extension code is the hard part. The Tachimanga/Suwayomi path provides evidence for transformed-JAR plus JVM/AndroidCompat. It requires a no-JIT JVM on iOS ARM, a compatible class library, and a host bridge.

### E — Script runtime

JavaScriptCore evaluates JavaScript and bridges native objects; [JavaScriptCore](https://developer.apple.com/documentation/javascriptcore) is official Apple API. It does not provide a browser DOM, fetch, Node, or unrestricted filesystem by itself. The host must provide HTTP/cookies/DOM/JSON/crypto/preferences/models. This is a strong safe fallback, not unchanged Kotlin compatibility.

### F — Native adapters

Native Swift preserves source identity, repository/update UX, preferences, and reliable behavior. It is best for initial sources but not ecosystem compatibility unless generated/maintained.

### G — Hybrid

Use JVM compatibility where proven, scripts for portable conversions, and native adapters for critical/exceptional sources. This is the least compromising path.

| Strategy | Compatibility | Complexity | Reliability | Security | Windows workflow | iOS viability |
|---|---:|---:|---:|---:|---:|---:|
| Native only | 0–2 | Low/medium | High | High | Good | High |
| Script only | 2–3 | Medium/high | Medium/high | High with bridge | Good | High |
| Translator | 3–4 for subset | High | Medium | Medium | Good | Medium |
| Embedded JVM | 4–4.5 if proven | Very high | Unknown/high risk | Low unless constrained | Mixed | Technically possible |
| Remote server | 4+ | Operationally high | Server-dependent | Server trust risk | Client easy | Violates local-first preference |
| Hybrid | 3 baseline; 4.5 conditional | High | High with fallback | Manageable | Good | Best overall |

## H. Recommended extension direction

Lock a host boundary, not one execution engine:

~~~text
ExtensionRepository
  -> verified ExtensionPackage
  -> ExtensionRuntimeRegistry
       -> JVMRuntime (feasibility-gated)
       -> ScriptRuntime
       -> NativeAdapterRuntime
  -> normalized SourceFacade
  -> downloads/player/reader/persistence
~~~

Execution policy:

1. Verify repository signature and artifact hash.
2. Run each source with timeout/cancellation boundaries.
3. Give scripts a narrow bridge for HTTP, cookies, DOM, crypto, preferences, logging, and models; never arbitrary filesystem/Keychain/native access.
4. Treat JVM sources as higher risk. Use classloader/bytecode restrictions and a deny-by-default host bridge where possible. If unrestricted Java APIs are unavoidable, mark Level 4.5 as a trust decision, not a sandbox guarantee.
5. Persist errors per extension/source so one failure cannot cancel all refreshes/downloads.
6. Preserve package/source IDs and source-local keys even for native adapters, keeping migration open.

Initial sources may be delivered as native adapters while the JVM experiment runs; that is sequencing, not abandonment of compatibility.

## I. Anime media and download findings

### I.1 Resolution forms found

The checked sources return direct MP4 or other file URLs, HLS master/media playlists, signed or expiring URLs, links requiring User-Agent/Referer/cookie/origin headers, nested hoster URLs, media discovered from inline JavaScript or AJAX JSON, VTT subtitle tracks, source-defined quality labels, and occasional torrent or FFmpeg argument metadata.

There is no universal “episode URL = file URL” assumption. Resolution belongs to the source and shared extractor layer; the native host should orchestrate and normalize the result.

### I.2 Quality policy

Normalize each candidate into numeric height when available, codec/container, bitrate/frame rate when available, source label, audio/subtitle language, direct/HLS/DASH kind, request headers/cookie reference, and stable resolver metadata. Do not persist only an expiring URL.

Ranking should implement the locked tier policy:

1. 1080p or higher when available;
2. otherwise the highest available source quality;
3. otherwise 720p;
4. otherwise 480p;
5. then the best remaining candidate.

Labels such as FHD, HD, 1080p HEVC, and 720p need parser normalization and fixtures. Keep a manual per-episode override.

### I.3 Download state machine

~~~text
queued -> resolving -> candidateReady -> transferring -> verifying -> complete
                         |              |              |
                         v              v              v
                   needsLogin       paused         failed/retryable
                         |
                         v
                   needsResolution
~~~

Persist title/episode/source IDs, selected resolver descriptor and quality, headers by reference, cookie/session reference, URL expiry/last resolution time, URLSession task/session identifiers, temporary/final paths, bytes/progress, expected length/checksum, attempts, retry-after, and verification result.

### I.4 Direct files

Use a background URLSession download task for direct HTTP(S) files. Apple documents that background sessions run out-of-process while an app is suspended, can relaunch the app for completion events, require a delegate/session identifier, and are rate-limited: [Downloading files in the background](https://developer.apple.com/documentation/foundation/downloading-files-in-the-background). Only HTTP/HTTPS and file-backed transfers are supported in this mode.

Write to a temporary file, verify length/container, then atomically move into the persistent download directory. Background URLSession resumption is more reliable than assuming resume data is always reusable; [Pausing and resuming downloads](https://developer.apple.com/documentation/foundation/pausing-and-resuming-downloads) documents server/protocol limitations.

### I.5 HLS

Use AVAssetDownloadURLSession for ordinary HLS offline assets where the playlist, segments, keys, and headers are compatible with Apple’s APIs; see [AVAssetDownloadURLSession](https://developer.apple.com/documentation/avfoundation/avassetdownloadurlsession) and [Offline playback and storage](https://developer.apple.com/documentation/avfoundation/offline-playback-and-storage).

Do not assume every source HLS stream works with AVAssetDownload. Signed segment URLs, per-request Referer/cookies, external subtitles, unusual codecs, and rapidly expiring tokens require a device experiment. The fallback may need foreground URLSession segment materialization and remuxing; that cannot promise unlimited background execution.

If a source URL expires while the app is suspended, the app cannot run the extension resolver at that moment. Mark the transfer needsResolution, re-run resolution when active/relaunched, and create a new transfer. Never treat a signed URL as stable identity.

### I.6 iOS 26 continuous processing

Modern iOS adds BGContinuedProcessingTask, a user-started task that may continue network/CPU work after the app is backgrounded and reports progress in a system UI; see [Apple’s long-running task guide](https://developer.apple.com/documentation/BackgroundTasks/performing-long-running-tasks-on-ios-and-ipados) and [BGContinuedProcessingTaskRequest](https://developer.apple.com/documentation/backgroundtasks/bgcontinuedprocessingtaskrequest?changes=_3__2). It must begin from an explicit user action, can be cancelled/expired, and is not guaranteed after force quit. Use it as a supplement for user-started materialization/remux work, not as an always-on queue.

### I.7 Wi-Fi and storage policy

Map Wi-Fi-only to URLSession cellular/expensive-network policy plus a current-path check. A user override can start cellular after confirmation. Persist the policy with each queue item.

Separate:

~~~text
Application Support/Downloads  persistent verified anime/manga files
Library/Application Support    database and migrations
Library/Caches                 image/thumbnails/resolver cache
tmp                            partial transfers and remux scratch
Documents/Backups              user exports
Logs                           bounded redacted diagnostics
~~~

Persistent downloads must not be purgeable cache. Automatic removal runs only when explicitly enabled and should consider completed/watch/read state rather than age alone.

## J. Player findings

AVPlayer/AVFoundation is appropriate for local playback and HLS. AVKit supplies PiP integration; Apple documents custom-player PiP in [Adopting Picture in Picture in a custom player](https://developer.apple.com/documentation/avkit/adopting-picture-in-picture-in-a-custom-player). Media-selection APIs expose embedded subtitle and alternative audio tracks; see [Selecting subtitles and alternative audio tracks](https://developer.apple.com/documentation/avfoundation/selecting-subtitles-and-alternative-audio-tracks).

Host-owned custom behavior includes gesture seek/skip, brightness/volume gestures, orientation lock, screen/control lock, exact position persistence, automatic next episode, resume action, custom speed controls, and optional external-player/share fallback.

AVPlayer rate is floating point, but Apple warns that rates above 2x depend on asset/device support and may degrade; [QA1772](https://developer.apple.com/library/archive/qa/qa1772/_index.html) confirms that 2.5x needs actual-asset testing. Treat 2.5x as an acceptance test, not an unconditional guarantee.

Do not claim universal MKV/WebM/AV1 support. The Android downloader often remuxes with FFmpeg and may produce MKV. iOS support depends on container/codec/tracks. Probe the asset, materialize a compatible file where possible, preserve the original verified file, and expose UIDocumentInteractionController Open In/share for unsupported files. Apple documents that controller as the system interface for previewing/opening unsupported files: [UIDocumentInteractionController](https://developer.apple.com/documentation/uikit/uidocumentinteractioncontroller). VLC/Infuse remain optional; discover installed handlers instead of making them dependencies.

Write playback position periodically and on pause/seek/background/end with explicit database saves. Automatic next episode must use source-scoped episode order.

## K. Manga findings

The manga source API maps naturally to iOS after removing Android URI/context types: source-scoped title identity, chapter number/name/URL/date/group, page list/order/headers, filters, preferences, and page URL re-resolution.

Use separate presentation models for:

1. paged reading (LTR/RTL, tap zones/swipes, zoom, orientation lock, fit width, optional double-page);
2. continuous/webtoon reading (vertical strip, preloading, long-image handling).

Store exact chapter and page progress. Keep image ordering independent of filesystem names. Prefetch a bounded number of pages and cancel prefetch when direction changes.

The Android reference writes zero-padded page names, MIME-sniffed extensions, optional ComicInfo.xml, and a temporary CBZ before atomic rename. The iOS equivalent can use a stable chapter folder or CBZ:

~~~text
Downloads/manga/<titleUUID>/<chapterUUID>/
  manifest.json
  0001.jpg
  0002.webp
  ...
~~~

CBZ is useful interchange; a folder plus manifest is easier for partial recovery. Local ZIP/CBZ import should copy/materialize into the app container unless a deliberate security-scoped bookmark policy is implemented. Tachimanga’s local-source guide confirms folder, ZIP/CBZ, RAR/CBR, and EPUB support, with folders/ZIP/CBZ preferred: [Tachimanga local source](https://tachimanga.app/help/guides/local-source.html).

## L. Persistence and storage findings

### L.1 Database

SwiftData-first is suitable for this personal app if it uses explicit identity constraints and migrations. Apple documents persistent models, ModelActor, and versioned schemas in [SwiftData persistence](https://developer.apple.com/documentation/swiftdata/preserving-your-apps-model-data-across-launches), [ModelActor](https://developer.apple.com/documentation/swiftdata/modelactor), [VersionedSchema](https://developer.apple.com/documentation/swiftdata/versionedschema), and [Unique constraints](https://developer.apple.com/documentation/swiftdata/unique%28_%3A%29).

Use one model-actor boundary for writes and explicit saves for progress/download transitions. If a prototype shows migration/index/transaction problems, GRDB/SQLite is a justified fallback; do not add it speculatively.

Recommended entities:

- Source and ExtensionPackage;
- SourcePreference;
- AnimeTitle and MangaTitle;
- AnimeEpisode and MangaChapter;
- EpisodeMediaCandidate and resolver descriptor;
- PageDescriptor;
- LibraryEntry and category/status;
- AnimeWatchProgress and MangaReadProgress;
- separate AnimeHistory and MangaHistory;
- DownloadRecord, DownloadFile, and DownloadTask;
- BackupRecord, DiagnosticEvent, and StoragePolicy.

### L.2 Stable identity

Upstream queries titles by source ID plus source-local URL; SQLDelight files getAnimeByUrlAndSource and getMangaByUrlAndSource demonstrate this. Episode/chapter synchronization matches URL and preserves progress/bookmark state when metadata changes.

Use:

~~~text
domain + sourceID + canonical sourceLocalKey
~~~

The key may be a URL, API UUID, or normalized stable ID. Do not merge same-looking titles across sources. Give local titles/episodes/chapters UUIDs for filesystem paths while retaining source ID/key for migration.

### L.3 Filesystem and counters

Use UUID directories with a manifest carrying source ID, source-local keys, display metadata, selected quality, media kind, checksum/size, and schema version. Renamed titles update metadata without renaming stable directories. Source URL changes use aliases/migrations.

Downloaded/unwatched/unread counters are derivable from complete download records and progress flags. Persist only if profiling needs it; otherwise use indexed queries.

### L.4 Diagnostics

Maintain bounded rotating local logs (OSLog plus redacted JSONL for export). Include extension/source ID, operation, timing, HTTP status, parse stage, download ID, and error class. Never log passwords, cookie contents, authorization headers, or complete signed tokens.

## M. Backup and migration findings

### M.1 Aniyomi backup

The checked-out Android BackupCreator writes a gzip-compressed Kotlin-Serialization Protocol Buffer file with a .tachibk suffix. It can include manga, anime, categories, source IDs, source preferences, app preferences, extension repository metadata, extension APKs, history, and progress. Media downloads are not embedded. Relevant paths are:

- aniyomi/app/src/main/java/eu/kanade/tachiyomi/data/backup/create/BackupCreator.kt
- aniyomi/app/src/main/java/eu/kanade/tachiyomi/data/backup/BackupDecoder.kt
- aniyomi/app/src/main/java/eu/kanade/tachiyomi/data/backup/models/Backup.kt
- BackupAnime.kt, BackupManga.kt, BackupEpisode.kt, and BackupChapter.kt in the same models directory.

Anime episodes carry exact lastSecondSeen and totalSeconds; manga chapters carry lastPageRead. Anime-specific fields use high Protocol Buffer numbers (500+), while standard manga fields remain in the Tachiyomi/Mihon lineage. An iOS importer can preserve source IDs/URLs and progress without restoring APK bytes.

### M.2 App-native backup

Recommend a versioned ZIP container:

~~~text
manifest.json
entities/anime.json
entities/manga.json
entities/progress.json
entities/history.json
entities/categories.json
entities/sources.json
entities/extensions.json
entities/preferences.json
checksums.json
~~~

Normal backup excludes media. An optional selected-download archive can be added later. Restore must validate manifest/checksums, stage records/files, map source IDs/keys, and commit transactionally so an invalid backup cannot partially overwrite the library.

### M.3 Tachimanga

Tachimanga’s official backup guide says backups include titles, categories, read chapters, history, extensions, settings, and related data, while downloads are excluded: [Tachimanga backups](https://tachimanga.app/help/guides/backups.html). Its changelog documents a native .tmb extension and a Tachiyomi-compatible backup mode: [Tachimanga changelog](https://tachimanga.app/docs/changelogs.html).

The internal native .tmb schema, compression details, and any encryption are **unknown** without a legitimate user-exported sample. Do not reverse protected data. A future migration path should accept the official Tachiyomi-compatible export first, then a documented/legitimate .tmb importer, with manual source/title mapping when IDs/URLs no longer resolve.

Tachimanga’s sync guide excludes local extensions, downloads, updates, settings, and tracker auth info from cloud sync, reinforcing a local-file migration path: [Tachimanga sync](https://tachimanga.app/help/guides/sync.html).

### M.4 Future Aniyomi migration

Preserving Aniyomi source IDs, package IDs, source-local URLs, episode/chapter URLs, source preferences, and progress makes a future importer cheap. Do not make it an MVP dependency.

## N. Windows-first iOS development workflow

### N.1 Windows-capable work

Swift’s Windows toolchain and Swift Package Manager can build/test platform-neutral packages; see [Swift for Windows](https://www.swift.org/install/windows/) and [Swift with VS Code](https://www.swift.org/documentation/articles/getting-started-with-vscode-swift.html). Windows can host:

- source protocol/model packages;
- quality ranking and URL canonicalization;
- backup DTO/manifest/fixture tests;
- source parser tests with captured HTML/JSON;
- filesystem layout/manifest tests;
- extension index/signature parsing;
- script translation tooling;
- Java/JVM and JAR transformation experiments;
- Git and CI orchestration.

### N.2 Mac/Xcode-required work

UIKit/SwiftUI/SwiftData/AVFoundation/AVKit/WebKit compilation against the iOS SDK, simulator execution, device deployment, signing, Instruments, and physical iPhone debugging require Xcode on macOS. Check [Xcode system requirements](https://developer.apple.com/xcode/system-requirements) when implementation starts.

GitHub-hosted macOS runners can perform clean builds, unit tests, simulator tests, and archive validation; see [GitHub-hosted runners](https://docs.github.com/en/actions/reference/runners/github-hosted-runners). CI cannot replace interactive device debugging or Apple signing state.

### N.3 Signing and installation

Personal Team profiles currently allow 10 App IDs, 3 devices, 3 installed apps per device, and seven-day expiry. Use one app target; source artifacts must stay data/scripts/JARs inside the app sandbox and must never become separate native frameworks.

SideStore’s official documentation supports a Windows-assisted installation path and seven-day refresh, with VPN/Wi-Fi requirements and an app slot: [SideStore prerequisites](https://docs.sidestore.io/docs/installation/prerequisites), [SideStore installation](https://docs.sidestore.io/docs/installation/install), [SideStore FAQ](https://docs.sidestore.io/docs/faq). AltStore uses AltServer on a reachable Windows/Mac computer and also requires seven-day refresh: [AltServer](https://faq.altstore.io/altstore-classic/altserver), [AltStore FAQ](https://faq.altstore.io/altstore-classic/your-altstore).

Recommended pipeline:

~~~text
Windows editor/tests
    -> private Git repository
    -> GitHub macOS CI (Swift package + simulator/unit validation)
    -> Mac/remote-Mac Xcode archive and free-profile signing
    -> direct device install during development
    -> SideStore or AltStore for recurring personal refresh
~~~

The first device deployment and any signing failure are required environment milestones.

### N.4 OS version uncertainty

The reported iOS 26.5.5 was not independently verified in the workspace. Apple’s security-update list should be checked on the phone against the Xcode SDK matrix; [Apple security releases](https://support.apple.com/id-id/100100) showed nearby 26.x releases but the exact patch should be confirmed later. Target the actual supported modern SDK, not the unverified patch string.

## O. iOS technical restrictions, signing, and policy

### O.1 OS-level facts

Apple validates executable code signatures and executable memory pages at runtime. An app cannot download an arbitrary unsigned native dynamic library and make it executable as if it were part of the signed app. Sideloading changes distribution/provisioning, not kernel code-signing or sandbox rules.

An interpreter can execute data held in the app sandbox. JavaScriptCore is such an interpreter. The official OpenJDK iOS port demonstrates a JVM Zero interpreter on ARM64 because JIT generation is unavailable. The runtime itself must still be compiled, signed, bundled, and maintained as app code.

### O.2 App Store policy versus OS

App Review Guideline 2.5.2 separately restricts downloading/installing/executing code that changes app functionality; see [App Store Review Guidelines](https://developer.apple.com/app-store/review/guidelines/). That is a distribution/review policy, not proof that the kernel cannot interpret data. Personal sideloading does not grant unrestricted native executable memory or a second native app.

Do not confuse JavaScriptCore/interpreted data with permission to download native code. All proposed runtimes must remain interpreted/data execution behind a narrow host bridge.

### O.3 Security

Use repository trust, artifact hash/signature, package/source IDs, capability declarations, and manual trust overrides for personal sources. For scripts, default-deny filesystem/Keychain/native calls and limit network to declared/user-approved domains. Enforce timeouts, cancellation, response-size/DOM-size limits, and memory budgets.

For a JVM runtime, a perfect in-process sandbox is unlikely. Classloader restrictions, bytecode scanning, API deny lists, and host-owned HTTP/file bridges reduce risk but do not formally isolate arbitrary JVM code. A source can still crash or exhaust the app. Proposal must state this trade-off.

## P. Architecture candidates

### Candidate 1 — Native Swift adapters

SwiftUI + SwiftData + URLSession + AVFoundation/AVKit + native source adapters. Best reliability/testability and Windows/CI portability. Reaches Level 0–2 and requires manual source maintenance.

### Candidate 2 — JavaScriptCore source runtime

Sources are JavaScript/TypeScript-like packages with a documented HTTP/DOM/preferences/model bridge. Good iOS fit and security control. Existing Kotlin extensions need translation/manual port. Reaches Level 2–3.

### Candidate 3 — Embedded JVM compatibility runtime

Bundle a no-JIT JVM interpreter, AndroidCompat/API shims, JAR/DEX transformation, class loader, and host bridge. Potentially reaches Level 4–4.5. Largest binary/maintenance/performance/security risk; requires a concrete prototype.

### Candidate 4 — Hybrid runtime-pluggable host (recommended)

Native Swift host and normalized domain protocol; native adapters for initial sources; JSC for converted/simple sources; embedded JVM path as a time-boxed compatibility capability. This preserves the locked goal as far as evidence allows without making the entire app depend on an unverified runtime.

| Dimension | Native | Script | JVM | Hybrid |
|---|---:|---:|---:|---:|
| Existing extension compatibility | Low | Medium after conversion | Highest if proven | Highest achievable with fallback |
| Offline anime/manga integration | High | High | Medium | High |
| Source failure isolation | High | Medium/high | Low/medium | High at host boundary |
| Windows development | High | High | Medium | High |
| iOS runtime risk | Low | Medium | Critical/high | Medium/high, isolated |
| Long-term maintenance | Medium manual | Medium | Very high | High but bounded |
| Security reviewability | High | High with bridge | Low/medium | Medium |

## Q. Testability and validation investigation

The architecture should expose deterministic seams:

- repository/package fixtures for index, signature/hash, version/API range, artifact metadata, and trust;
- identical runtime-contract tests against native, script, and JVM implementations;
- captured HTML/JSON fixtures for title/detail/episode/chapter/page/media parsing;
- URLRequest tests for headers, cookies, redirects, timeout/cancellation, response limits, and expiry;
- direct MP4/HLS media-resolution fixtures, quality ranking, VTT tracks, and unsupported-container classification;
- download state tests for retry, pause/resume, partial files, process restart, atomic rename, checksum, Wi-Fi policy, and needsResolution;
- SwiftData migration/uniqueness/progress-preservation fixtures and orphan-file reconciliation;
- backup validation, corrupted archive handling, source mapping, exact seconds/page progress, idempotent restore;
- reader/player tests for page ordering, prefetch cancellation, position writes, local AVAsset, tracks, PiP, 2.5x acceptance, and Open In;
- failure-isolation tests proving one source timeout/exception cannot cancel unrelated refresh/download work.

Use recorded fixtures for normal tests; live websites should only be smoke tests because HTML, signed URLs, and anti-bot behavior change independently.

## R. Risk register

| Risk | Rank | Evidence/impact | Mitigation | Proposal treatment |
|---|---|---|---|---|
| Existing Aniyomi APK/JAR compatibility | Critical | Android loader needs Dalvik/PackageManager; Tachimanga proves a private JVM path but not reproduction | No-JIT JVM spike plus native/script fallback | Explicit gate and acceptance matrix |
| Dynamic executable/JIT restrictions | Critical | Apple signed executable pages; OpenJDK Zero interpreter required on iOS ARM | Interpret data only; bundle signed runtime; no JIT | Runtime/security section |
| JVM AndroidCompat surface | Critical | Context, preferences, Uri, Handler, WebView, Rx, OkHttp, Jsoup imports | Minimum API, corpus analyzer, capability declarations | Required before Level 4.5 claim |
| Anime extractor complexity | High | AnimeKai/MegaUp uses nested hosts, decode calls, HLS, headers, VTT, Android side effects | Native extractor interface and resolver fixtures | Anime media spec |
| HLS/offline materialization | High | AVAssetDownload only covers compatible HLS | Device spike; direct/HLS state machines; foreground fallback | Downloader design |
| Expiring URLs | High | MangaDex at-home and AnimeKai links are temporary | Stable resolver keys; needsResolution | Source/download contract |
| Background execution | High | URLSession is system-managed; force quit/cancellation/ratelimits remain | URLSession background plus optional iOS26 task | Platform constraints |
| AVPlayer codec/container mismatch | High | Android reference remuxes; AVPlayer support depends on asset | Probe/materialize/Open In | Player acceptance |
| Source breakage | High | Website HTML/API changes are source-owned | Versioned packages, per-source isolation, fixtures/logs | Extension manager |
| Free signing | High | Seven-day profiles and app/device slots | One target, CI archive, Mac milestone, SideStore/AltStore | Workflow |
| Windows assumptions | High | Swift/iOS SDK/Xcode need Mac final stage | Platform-neutral packages + CI + remote Mac | Workflow |
| Tachimanga .tmb unknown | Medium/high | Format documented, schema not public | Legitimate sample; Tachiyomi-compatible first | Migration |
| SwiftData migration/concurrency | Medium | Modern API exists but device behavior needs testing | ModelActor, VersionedSchema, explicit saves; fallback | Persistence |
| Extension trust/security | Medium/high | Downloaded code may be malicious/unstable | signatures, capability bridge, timeouts, redacted logs | Security |
| Source ID/API drift | Medium | Anime 12–16 and manga 1.4–1.6 vary | Host API table, adapters, aliases | Protocol |
| Storage growth | Medium | Reliability over storage efficiency | accounting, limits, explicit cleanup | Storage |

## S. Open questions remaining

### Technical questions requiring a prototype or artifact

1. Can a modern iOS SDK build/run a minimal no-JIT JVM/Zero interpreter with dynamic JAR classes under a Personal Team profile?
2. Can it load a current lib 1.6 MangaDex JAR and an Aniyomi anime JAR with Kotlin coroutines, serialization, OkHttp, Jsoup, and AndroidCompat?
3. Does the current JAR artifact contain sufficient manifest/resources/assets, or is APK conversion required?
4. Which Android framework classes are needed by the initial sources after static dependency analysis?
5. Can SwiftSoup match the Jsoup selectors/DOM behavior in representative sources?
6. Can AVAssetDownload handle representative HLS headers, cookies, subtitles, and expiring URLs on the target iPhone?
7. Which downloaded containers/codecs play locally and which require remux/Open In?
8. Are the relevant background capabilities available under a free Personal Team profile?
9. What is the internal native Tachimanga .tmb schema in a legitimate export?
10. What percentage of a deliberately selected extension corpus passes runtime/parser tests? No defensible ecosystem percentage follows from three samples.

### User-choice questions

No locked product requirement needs to be reopened. The remaining choices are implementation gates and can be carried into Proposal. The only possible future confirmation is whether to accept a trust warning for an unrestricted JVM compatibility runtime if strong in-process sandboxing cannot be achieved; it must not be silently assumed.

## T. Recommended pre-Proposal decisions

| Decision | Options | Recommendation | Consequence of postponing |
|---|---|---|---|
| Extension execution boundary | Native-only, script-only, embedded JVM, runtime-pluggable | Runtime-pluggable host with JVM feasibility gate and native/script fallback | Premature choice could discard highest compatibility |
| Initial compatibility promise | Level 2, Level 3, conditional Level 4.5 | Promise Level 2/3; label Level 4.5 experimental until device proof | Overpromising makes Proposal false |
| Source identity | Title-only, source+URL, source+stable key | Domain + source ID + source-local key, with local UUIDs | Migration/downloads become fragile |
| Persistence | SwiftData, SQLite/GRDB, filesystem-only | SwiftData first with ModelActor/VersionedSchema; filesystem for media | Premature database switch adds dependency |
| Backup | JSON, gzip protobuf, ZIP manifest | Versioned ZIP with JSON/entities/checksums; separate Aniyomi importer | Schema evolution becomes coupled to media |
| Download architecture | Streaming-first, direct-only, unified | Offline-first coordinator with separate direct/HLS materializers | Signed/HLS sources are mis-modelled |
| Trust | Unrestricted code, repo trust, no installs | Signed/hash-verified packages and capability bridge | Security/failure isolation becomes untestable |
| iOS target | Reported patch, old compatibility, modern verified OS | Verify phone/Xcode SDK, then target supported modern OS | Unusual patch string could invalidate build |

No additional user-level product decisions are required before /opsx:propose. Proposal should contain these technical gates and must not silently downgrade extension, offline-download, manga, migration, or reliability requirements.

# READY FOR OPENSPEC PROPOSAL

The evidence supports taking this direction into Proposal:

1. Native Swift/SwiftUI host with SwiftData-or-equivalent local persistence, source-scoped identities, separate anime/manga histories, durable filesystem downloads, bounded diagnostics, and versioned local backup.
2. Normalized source protocol covering search/browse/details, episodes/chapters, media/page descriptors, filters, preferences, source IDs, and stable source-local keys.
3. Extension repository/installer/trust subsystem preserving Aniyomi/Tachiyomi package IDs, source IDs, metadata, versions, update state, and preferences without installing native executable extensions.
4. Runtime registry with mandatory no-JIT JVM feasibility spike, constrained JavaScript/portable-script runtime, and native Swift adapters. Runtime contract tests must make fallback first-class.
5. Anime-first offline pipeline with direct-file URLSession, HLS strategy, quality normalization, headers/cookies, expiry re-resolution, verification, atomic files, retries, pause/cancel, storage limits, Wi-Fi policy, and background limitations.
6. Custom AVPlayer/AVKit player with exact local progress, PiP, 2.5x acceptance tests, gestures, native track support, and Open In/share fallback.
7. Manga pipeline with source-specific chapter/page extraction, prefetch, RTL/LTR/vertical modes, exact page progress, folder/CBZ offline storage, local ZIP/CBZ import, and expiry re-resolution.
8. Migration seams for Aniyomi/Tachiyomi-compatible backups and a future Tachimanga .tmb importer after a legitimate sample.
9. Windows-first repository/CI workflow with an explicit Mac/Xcode signing milestone and free-Personal-Team/SideStore constraints.
10. Security/failure model treating downloaded extensions as untrusted code/data, isolating failures as iOS permits, redacting diagnostics, and separating OS enforcement from App Store policy.

The Proposal must not forget:

- the Tachimanga/Suwayomi JVM counterexample and the need to reproduce or rule it out;
- extension library version drift;
- HLS/direct media differences and expiring URLs;
- source+title identity separation;
- no automatic background library notifications;
- persistent downloads versus purgeable caches;
- exact anime seconds and manga page progress;
- repository trust/update/disable/uninstall UX;
- deterministic fixtures and source failure diagnostics;
- manual local backup/restore without cloud accounts;
- the target repository is currently empty and upstream Android remains reference-only.
