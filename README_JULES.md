# README_JULES.md — Build Plan for **LAN Drop** (Flutter · Clean Architecture · BLoC)

> This README tells **Google Jules** exactly how to implement our cross-platform, local-network (same Wi-Fi/LAN) **file transfer + chat** app with a 3-tab UI (Known / Nearby / Settings), DNS-SD (Bonjour/NSD) discovery, and TCP/WebRTC transports. Jules will read this file, develop a plan, clone the repo in a secure Cloud VM, run the work, and open **Pull Requests** that show its plan, reasoning, and diffs. ([Jules][1])

---

## 0) How Jules fits this repo

* **Workflow**: File issues in this repo; Jules clones the code into a **Cloud VM**, executes a step-by-step plan, and opens a PR with diffs and an explanation (“shows work”). It can be targeted by labeling issues (e.g., `jules`). ([Jules][1])
* **Access**: Sign in at **jules.google.com** with your Google account and connect GitHub. ([Jules][2])
* **Availability & plans**: Jules is publicly available; Google describes it as an asynchronous coding agent powered by **Gemini 2.5 Pro** with a quality “critic” stage; press coverage notes free and paid tiers. ([blog.google][3])

---

## 1) Product summary — **LAN Drop**

A cross-platform utility for **discovering peers on the same LAN**, **chatting**, and **sending files** with a chat-style UI.

* **Discovery**: **DNS-SD (Bonjour/NSD)** service `_lanshare._tcp`. We advertise TXT: `name`, `uuid`, `platform`, `port`. Flutter choices: `multicast_dns` (pure Dart) or `flutter_nsd` (platform NSD/Bonjour). ([Dart packages][4])
* **Transport**:

  * **Mobile/Desktop** — **TCP sockets** via `dart:io` (`ServerSocket`/`Socket`). ([Flutter API Docs][5])
  * **Web (optional)** — **WebRTC DataChannels** (browsers can’t use `dart:io`). Plugin: `flutter_webrtc`. ([Dart packages][6])
* **UI**:

  1. **Known** (saved peers: name/IP/UUID/last-seen)
  2. **Nearby** (live discovery list: name/IP/UUID)
  3. **Settings** (display name, advertise toggle, default download folder, firewall/iOS permission tips)
* **State management**: **BLoC** (`flutter_bloc`/`bloc`), immutable states. ([Dart packages][7])

### Platform policy requirements (to be implemented)

* **iOS/iPadOS**: Add **`NSLocalNetworkUsageDescription`** and list Bonjour service types in **`NSBonjourServices`** (Local Network privacy). ([Apple Developer][8])
* **Android 13+**: Use the **`NEARBY_WIFI_DEVICES`** runtime permission for nearby Wi-Fi operations (supersedes location for these use cases). ([Android Developers][9])

---

## 2) Engineering constraints

* **Framework**: Flutter for iOS/Android/Windows/macOS; optional web.
* **Architecture**: **Clean Architecture** — Presentation → Domain → Data; dependencies point inward.
* **Discovery**: DNS-SD only (Bonjour/NSD), no raw UDP broadcast.
* **Transport**: TCP on mobile/desktop; **WebRTC** on web; **never import `dart:io` in web**.
* **Testing**: High coverage on domain and BLoC; desktop loopback integration tests for TCP.

---

## 3) Repository layout (to be scaffolded by Jules)

> Feature-first, Clean Architecture inside each feature; core and platform adapters are separated. Monorepo-friendly (Pub Workspaces / Melos).

```
.
├─ WORKSPACE.md                   # how to run the monorepo (pub workspaces / melos)
├─ melos.yaml                     # optional: melos scripts for bootstrap, tests, quality
├─ pubspec.yaml                   # root workspace; lists all packages/apps as path deps
├─ tools/                         # repo-level scripts (codegen, format, coverage)
│
├─ packages/
│  ├─ core/                       # stable, pure Dart building blocks (NO Flutter here)
│  │  ├─ core_domain/             # value objects, failures, cross-cutting entities
│  │  ├─ core_usecases/           # base UseCase types, Result/Either helpers
│  │  ├─ core_protocol/           # message headers, chunk frames, (de)serializers
│  │  └─ core_testing/            # fakes/mocks, golden utils shared across tests
│  │
│  ├─ platform/                   # adapters that touch OS/frameworks (outer circle)
│  │  ├─ discovery_dns_sd/        # DNS-SD (Bonjour/NSD) adapter (multicast_dns/flutter_nsd)
│  │  ├─ transport_tcp/           # dart:io Socket/ServerSocket (mobile/desktop)
│  │  ├─ transport_webrtc/        # WebRTC DataChannels (web & optional native)
│  │  └─ persistence_kv/          # key-value storage (settings, known peers)
│  │
│  └─ features/                   # each feature is independently shippable & testable
│     ├─ peers/                   # "Known" + "Nearby" lists
│     │  ├─ domain/               # ← inner: pure Dart
│     │  │  ├─ entities/          # Peer, Endpoint
│     │  │  ├─ repositories/      # abstract contracts (DiscoveryRepository, PeersRepo)
│     │  │  └─ usecases/          # StartAdvertising, BrowsePeers, PersistKnownPeers…
│     │  ├─ data/                 # ← outer: implementations depend on platform/*
│     │  │  ├─ mappers/           # DTO ↔ Entity
│     │  │  ├─ datasources/       # dnsSdBrowser, dnsSdAdvertiser
│     │  │  └─ repositories/      # PeersRepoImpl (binds datasources → domain contracts)
│     │  └─ presentation/         # Flutter/BLoC only here
│     │     ├─ bloc/              # NearbyBloc, KnownBloc (+ events/states)
│     │     ├─ pages/             # KnownPage, NearbyPage
│     │     ├─ widgets/           # cards, list items
│     │     └─ routes.dart
│     │
│     ├─ thread/                  # chat + file transfer thread
│     │  ├─ domain/
│     │  │  ├─ entities/          # Message, Transfer
│     │  │  ├─ repositories/      # TransportRepository, ThreadRepository
│     │  │  └─ usecases/          # SendText, SendFile, ResumeTransfer, PairDevice
│     │  ├─ data/
│     │  │  ├─ mappers/
│     │  │  ├─ datasources/       # tcpClient, tcpServer, webrtcChannel (behind interfaces)
│     │  │  └─ repositories/      # ThreadRepoImpl (chooses TCP vs WebRTC via DI)
│     │  └─ presentation/
│     │     ├─ bloc/              # ThreadBloc (progress, errors, acks)
│     │     ├─ pages/             # ThreadPage
│     │     └─ widgets/           # bubbles, file cards, progress bars
│     │
│     └─ settings/                # profile name, advertise toggle, folders, tips
│        ├─ domain/
│        │  ├─ entities/          # AppSettings
│        │  ├─ repositories/      # SettingsRepository
│        │  └─ usecases/          # UpdateDisplayName, ToggleAdvertise, GetSettings
│        ├─ data/
│        │  ├─ mappers/
│        │  ├─ datasources/       # kvStore, platform permission helpers
│        │  └─ repositories/      # SettingsRepoImpl
│        └─ presentation/
│           ├─ cubit/             # SettingsCubit
│           ├─ pages/             # SettingsPage
│           └─ widgets/
│
├─ app/                           # executable Flutter app (composition root)
│  ├─ lib/
│  │  ├─ app.dart                 # MaterialApp, navigation shell
│  │  ├─ di/                      # get_it wiring of feature/domain/platform packages
│  │  ├─ theme/                   # light/dark themes, typography
│  │  ├─ routing/                 # GoRouter/Router config stitching features
│  │  └─ bootstrap.dart           # runZonedGuarded, BlocObserver, env setup
│  └─ test/                       # app-level widget/golden tests
│
└─ tests/
   ├─ unit/                       # package-scoped unit tests (or colocate in packages/*)
   ├─ widget/                     # shared widget/golden helpers
   └─ integration/                # E2E: loopback TCP 50MB, WebRTC round-trip, DNS-SD smoke
```

**Notes for Jules (scaffolding rules)**

* Each package has its own `pubspec.yaml` and tests; root `pubspec.yaml` defines a **pub workspace**.
* Use **conditional imports** so **web builds never import `dart:io`**; web uses the WebRTC transport package. ([Dart packages][6])
* Presentation may import `flutter_bloc`; **domain & data stay Flutter-agnostic**. ([Dart packages][7])

---

## 4) Milestones & issues (create one GitHub issue per milestone)

> Each issue must include: **Context**, **Requirements**, **Acceptance Criteria**, **Definition of Done**. Jules will propose a plan, execute it in a VM, and open a PR that shows the plan/diff. ([Jules][1])

### M0 — Bootstrap repo, CI, lint

**Context**: Start project skeleton.
**Requirements**

* Scaffold the layout above.
* Add dependencies: `flutter_bloc`, `bloc`, `equatable`, `uuid`, `file_picker`, `path_provider`, `multicast_dns` **or** `flutter_nsd`, optional `flutter_webrtc`, DI (`get_it`), lints (`flutter_lints`). ([Dart packages][7])
* GitHub Actions: `flutter analyze`, `flutter test`, debug build.
* Hello-world Flutter screen.
  **Acceptance Criteria**: CI green; sample widget test passes.
  **DoD**: PR includes plan + diffs + local run instructions.

### M1 — Domain layer (entities & use cases)

**Requirements**

* Entities: `Peer {uuid, name, ip, port, platform}`, `Message`, `Transfer`, `Endpoint`.
* Use cases: `StartAdvertising`, `BrowsePeers`, `SendText`, `SendFile`, `ResumeTransfer`, `PairDevice`, `PersistKnownPeers`, `UpdateDisplayName`.
* 100% unit coverage for domain (pure Dart).
  **DoD**: No Flutter or platform imports.

### M2 — Discovery adapter (DNS-SD)

**Requirements**

* Implement `DiscoveryRepository` in `/packages/platform/discovery_dns_sd` using **DNS-SD** (Bonjour/NSD) with either `multicast_dns` (pure Dart) or `flutter_nsd` (platform). ([Dart packages][4])
* Advertise `_lanshare._tcp` with TXT: `name`, `uuid`, `platform`, `port`.
* Browse & resolve peers; emit `Stream<Peer>` (add/remove/update).
  **Acceptance Criteria**: Demo Nearby screen shows live devices (mockable).
  **DoD**: Unit tests simulating add/remove/resolve events.

### M3 — TCP transport (mobile/desktop)

**Requirements**

* Implement `TransportRepository` in `/packages/platform/transport_tcp` using `ServerSocket.bind(InternetAddress.anyIPv4, 0)` for dynamic port; sender uses `Socket.connect(ip, port)`. ([Flutter API Docs][5])
* Protocol: JSON header (`text`/`file`), file chunking (64–256 KB) with `seq`, `offset`, `len`, optional CRC; resume via last offset request.
* Loopback tests for a 50 MB stream (bounded memory, backpressure).
  **Acceptance Criteria**: Thread demo can send text + a file between two local endpoints (desktop CI).
  **DoD**: PR includes throughput notes and backpressure approach.

### M4 — WebRTC transport (web optional)

**Requirements**

* Implement `/packages/platform/transport_webrtc` with `flutter_webrtc` **DataChannel** (reliable mode), reusing the same header/chunk protocol. ([Dart packages][6])
* **Conditional imports**: web uses WebRTC transport; mobile/desktop use TCP (**never import `dart:io` on web**).
* Minimal local signaling demo (manual peer IP/QR) to bring up a channel and echo bytes.
  **Acceptance Criteria**: A smoke test that round-trips a buffer on web.
  **DoD**: CI job that builds web target and runs the smoke test.

### M5 — Persistence & settings

**Requirements**

* Implement key-value storage for Known peers, display name, advertise toggle, default download folder (`/packages/platform/persistence_kv`).
* Wire Settings screen to values.
  **DoD**: Tests for save/load and migration.

### M6 — Presentation (BLoC) & 3 tabs

**Requirements**

* BLoCs: `NearbyBloc`, `KnownBloc`, `ThreadBloc`, `SettingsCubit`.
* Screens:

  * **Known**: saved peers → open thread.
  * **Nearby**: live discovery → tap to open thread or request pairing PIN.
  * **Settings**: edit name, advertising toggle, pick folder, platform tips.
* Widget tests for each screen; golden tests for main layouts.
  **DoD**: No plugin calls in Widgets; all side-effects via repositories/use cases. ([Dart packages][7])

### M7 — Thread screen (chat + file)

**Requirements**

* Chat bubbles (text), file cards (name/size/progress/ETA), cancel/resume.
* `ThreadBloc` exposes `TransferProgress` (bytes sent/total + rate).
  **DoD**: Reliable progress even if the user switches tabs (no background services in v1).

### M8 — Platform integrations & permissions

**Requirements**

* **iOS**: Add `NSLocalNetworkUsageDescription` (clear rationale) and **`NSBonjourServices`** including `_lanshare._tcp` to `Info.plist`. ([Apple Developer][8])
* **Android 13+**: Request **`NEARBY_WIFI_DEVICES`** at runtime for nearby Wi-Fi operations (avoid legacy location). ([Android Developers][9])
* **Windows**: First-run tip to allow the app on **Private** networks via Windows Firewall.
* **macOS (App Store sandbox)**: note network client/server entitlements if distributing there.
  **DoD**: Settings page includes a “Why these permissions” explainer.

### M9 — QA & hardening

**Requirements**

* Retries + backpressure; bounded memory on large files; sanitize filenames; write to temp and atomic move on completion.
* Desktop CI integration test: start local `ServerSocket`, transfer 50 MB, verify checksum; headless widget tests.
  **DoD**: CI green across analyze/test/build.

### M10 — Release docs & store readiness

**Requirements**

* Generate `README_POLICIES.md` summarizing platform keys/permissions and store listing notes (Android Data Safety, iOS rationale).
* Add PRIVACY.md (“LAN-only; no external servers”).
* Produce `RELEASE_CHECKLIST.md` (TestFlight, Play Console, desktop packaging).
  **DoD**: Docs linked from main README; sample screenshots added.

---

## 5) Definition of Done (global)

* PR must include **Jules plan & reasoning** and **diffs**, with instructions to run tests locally (Jules “shows work”). ([Jules][1])
* **Clean Architecture** boundaries respected (Presentation → Domain → Data).
* **Tests**: domain/use cases 100% coverage; overall coverage target ≥80%.
* CI: `flutter analyze`, `flutter test` (unit + widget), and at least one desktop TCP integration test must pass.

---

## 6) Issue template (paste into each GitHub Issue for Jules)

```
Title: <Milestone Title>

Context:
See README_JULES.md §<milestone>. We are building a LAN-only file share + chat
app with DNS-SD discovery and TCP/WebRTC transports in Flutter using Clean
Architecture + BLoC. Follow all architecture and policy constraints.

Requirements:
- <bullet list from the milestone>

Acceptance Criteria:
- <list of measurable outcomes and demos>

Definition of Done:
- PR with plan, reasoning, diffs, tests passing (CI green).
```

> Tip: apply the `jules` label so Jules picks it up. ([Jules][1])

---

## 7) Useful references for the agent & reviewers

* **Jules product pages / behavior**: how it clones to a Cloud VM, shows a plan, and opens PRs. ([Jules][1])
* **Jules announcement / model**: Gemini-powered asynchronous coding agent with a quality “critic” stage; GA/free-tier coverage. ([blog.google][3])
* **BLoC**: packages and official docs. ([Dart packages][7])
* **DNS-SD in Flutter**: `multicast_dns` and `flutter_nsd`. ([Dart packages][4])
* **Sockets**: `ServerSocket` / `Socket` in `dart:io`. ([Flutter API Docs][5])
* **WebRTC**: `flutter_webrtc` (DataChannels). ([Dart packages][6])
* **Platform policies**: iOS Local Network privacy keys; Android `NEARBY_WIFI_DEVICES`. ([Apple Developer][8])

---

**Copy-paste this file as `README_JULES.md` at the repo root**, then open Issue **M0** (“Bootstrap repo, CI, lint”) to start the process.

[1]: https://jules.google/?utm_source=chatgpt.com "Jules - An Asynchronous Coding Agent"
[2]: https://jules.google.com/?utm_source=chatgpt.com "Google Jules"
[3]: https://blog.google/technology/google-labs/jules/?utm_source=chatgpt.com "Jules: Google's autonomous AI coding agent"
[4]: https://pub.dev/packages/multicast_dns?utm_source=chatgpt.com "multicast_dns | Dart package"
[5]: https://api.flutter.dev/flutter/dart-io/ServerSocket-class.html?utm_source=chatgpt.com "ServerSocket class - dart:io library"
[6]: https://pub.dev/packages/flutter_webrtc?utm_source=chatgpt.com "flutter_webrtc | Flutter package"
[7]: https://pub.dev/packages/flutter_bloc?utm_source=chatgpt.com "flutter_bloc | Flutter package"
[8]: https://developer.apple.com/documentation/bundleresources/information-property-list/nslocalnetworkusagedescription?utm_source=chatgpt.com "NSLocalNetworkUsageDescription"
[9]: https://developer.android.com/develop/connectivity/wifi/wifi-permissions?utm_source=chatgpt.com "Request permission to access nearby Wi-Fi devices"