# README.md — LAN Drop (overview, design, and implementation guide)

## 1) Product idea (deep description)

**LAN Drop** is a cross-platform, local-network (“same Wi-Fi/LAN”) utility to **discover peers**, **chat**, and **send files** with a **chat-style UI**.

* **Discovery**: devices publish/resolve a Bonjour/NSD service (e.g. `_lanshare._tcp`) and show up in a **Nearby** tab with **name, IP, UUID**. Bonjour/DNS-SD is the industry standard for zero-config local discovery. ([Apple Developer][1])
* **Transport**: on mobile/desktop we use **TCP sockets** (`dart:io`) for simple, fast LAN transfers; on **web** (optional) we use **WebRTC DataChannels** because browsers don’t allow raw TCP/UDP. ([Dart API][2])
* **Identity**: each device advertises a persistent `uuid`, a human-editable `displayName`, and the listening `port`. The **IP** is shown from the active interface.
* **Security**: v1 supports a **pairing PIN** (optional). If you target the web build, **WebRTC DTLS** gives you encryption by default. ([MDN Web Docs][3])

### Why Flutter?

* One codebase for **iOS/Android/macOS/Windows/Linux**; **web** supported via WebRTC.
* Note: **Flutter web cannot use `dart:io` sockets**; use WebRTC (or skip web). ([Dart][4])

---

## 2) UX & navigation

**Tabs (BottomNav):**

1. **Known** — peers you’ve connected with before (name • IP • UUID • last seen). Tap → chat thread.
2. **Nearby** — live devices on LAN discovered via DNS-SD. Tap → open thread (or “Request Pair” if PIN enabled).
3. **Settings** — edit display name, toggle “Advertise presence”, default download folder, firewall/iOS permission help.

**Thread screen:**

* Header: peer name, status (Online/Offline/Resolving…).
* Message list: text bubbles + file cards (show filename, size, progress, speed).
* Composer: text input + “Attach file”.
* Transfer engine: chunked streaming (64–256 KB), resume by offset.

---

## 3) Architecture (Clean Architecture + BLoC)

**Layering (dependency rule points inward):**

* **Presentation**: Flutter UI widgets + `flutter_bloc` (Blocs/Cubits), pure rendering of states. ([Bloc][5])
* **Domain**: use cases (`DiscoverPeers`, `StartAdvertising`, `SendFile`, `SendText`, `ResumeTransfer`, `PairDevice`), entities (`Peer`, `Message`, `Transfer`), repositories’ **interfaces**.
* **Data**: repository **implementations** (DNS-SD adapter, sockets/WebRTC adapter, key-value store), data sources (platform plugins, disk), DTOs/mappers.

This follows Clean Architecture’s separation of concerns and testability principles. ([blog.cleancoder.com][6])

---

## 4) Project structure (monorepo)

```
/packages
  /core_domain/               # entities, value objects, failures
  /core_usecases/             # DiscoverPeers, SendFile, ...
  /core_protocol/             # message headers, chunk structs, serializers
  /infra_discovery/           # DNS-SD adapter (multicast_dns | flutter_nsd)
  /infra_transport_tcp/       # dart:io Socket + ServerSocket
  /infra_transport_webrtc/    # optional web/desktop/mobile via flutter_webrtc
  /infra_persistence/         # hive/shared_prefs or file-backed KV
/app
  /lib
    /features/peers/          # Blocs, pages, widgets
    /features/thread/
    /features/settings/
    /routing/
    /di/                      # get_it or provider of repositories/usecases
/test                         # unit tests for every package + widget tests
```

---

## 5) Discovery design (DNS-SD / Bonjour / NSD)

* **Service type**: `_lanshare._tcp`
* **TXT records**: `name`, `uuid`, `platform`, `proto=1`, `port=<listenPort>`

**Flutter options:**

* `multicast_dns` (pure Dart; perform PTR→SRV→A lookups), or
* `flutter_nsd` (wraps platform NSD/Bonjour; discovery only). ([Dart packages][7])

**Android**: NSD is the official API. ([Android Developers][8])
**Apple**: Bonjour is the official API. ([Apple Developer][1])

---

## 6) Transport design

### TCP (mobile/desktop)

* **Receiver**: `ServerSocket.bind(InternetAddress.anyIPv4, 0)`; advertise the chosen port in TXT; accept connections; route to per-peer handlers.
* **Sender**: `Socket.connect(ip, port)`; send a **JSON header** then raw bytes.
* Dart `Socket`/`ServerSocket` APIs are standard. ([Dart API][9])

**Header examples**

```json
{ "type":"text", "id":"msg-uuid", "ts": 1699999999, "body":"hello" }
{ "type":"file", "id":"msg-uuid", "ts": 1699999999, "name":"IMG_001.jpg", "size":1048576, "mime":"image/jpeg" }
```

**Chunk frame**

```json
{ "type":"chunk", "id":"msg-uuid", "seq":12, "offset":786432, "len":65536, "crc32":"..." }
```

**Resume**: ask peer `resume?id=msg-uuid` → returns last persisted `offset`.

### Web/optional (WebRTC DataChannel)

* DataChannels are **encrypted (DTLS)** and support reliable/unreliable modes; ideal for browser builds. Use `flutter_webrtc`. ([MDN Web Docs][3])

---

## 7) Permissions & platform specifics (summary)

* **iOS/iPadOS**: add `NSLocalNetworkUsageDescription` and **declare your Bonjour types** in `NSBonjourServices`. ([Apple Developer][10])

  * If you rely on **multicast/broadcast beyond DNS-SD**, you may need the **multicast entitlement** (`com.apple.developer.networking.multicast`). Prefer DNS-SD to avoid it. ([Apple Developer][11])
* **Android 12/13+**: use **`NEARBY_WIFI_DEVICES`** instead of location; add `neverForLocation` when applicable. ([Android Developers][12])
* **Windows**: inbound listener needs firewall allow on **Private network**. Provide an in-app tip. ([Microsoft Support][13])
* **Web**: `dart:io` is **not supported** → use WebRTC or WebSockets. ([Dart][4])

---

## 8) Step-by-step implementation (for Cursor)

1. **Boot**

   * Create monorepo packages above.
   * Add dependencies: `flutter_bloc`, `equatable`, `uuid`, `file_picker`, `path_provider`, `multicast_dns` or `flutter_nsd`, optional `flutter_webrtc`.

2. **Discovery adapter**

   * Implement `DiscoveryRepository` with `startAdvertising()`, `browse()`, `stop()` using DNS-SD. (On Android reference NSD docs for service browse/resolve; on Apple declare types.) ([Android Developers][8])

3. **Transport adapter (TCP)**

   * Implement `TransportRepository` with `listen()`, `connect(host,port)`, `sendText()`, `sendFile(stream,size)`, `resume(id,offset)`. Use `ServerSocket`/`Socket`. ([Flutter API Docs][14])

4. **WebRTC adapter (optional)**

   * Use `flutter_webrtc` DataChannel with a trivial in-LAN signaling (e.g., QR code or manual IP:port exchange) for browser targets. ([Dart packages][15])

5. **Use cases** (domain)

   * `DiscoverPeers`, `PersistKnownPeers`, `OpenThread`, `SendText`, `SendFile`, `CancelTransfer`, `ResumeTransfer`, `PairDevice`, `UpdateDisplayName`.

6. **BLoCs**

   * `NearbyBloc` (states: Idle/Discovering/Found/Errored).
   * `KnownBloc` (Load/Save).
   * `ThreadBloc` (SendText/File, Progress, Ack, Error).
   * `SettingsCubit` (name, advertise toggle, folder path).
     BLoC concepts & patterns per official docs. ([Bloc][5])

7. **UI**

   * 3 tabs + thread page. Use `BlocBuilder/BlocListener` to render/progress. ([Bloc][5])

8. **Platform files**

   * iOS: add `NSLocalNetworkUsageDescription` + `NSBonjourServices` to Info.plist before you ever browse/advertise. ([Apple Developer][10])
   * Android: declare/request `NEARBY_WIFI_DEVICES` for API 33+, and only if you truly need it. ([Android Developers][12])

9. **Testing**

   * Unit tests for use cases (pure Dart).
   * Mock repositories to simulate transfers and discovery.
   * Widget tests for tab flows and thread rendering.

---

## 9) Performance notes

* Use backpressure (pause the stream) when the receiver’s sink is busy.
* Default chunk size 128 KB; tune for Wi-Fi.
* Persist transfers to resume on app relaunch.

---

## 10) Future work

* End-to-end encryption with pre-shared key or ECDH during pairing.
* Multi-file queueing & drag-and-drop on desktop.
* Background transfers (platform-specific constraints apply).

---

# README_ENGINEERING.md — Engineering principles, Clean Architecture & BLoC rules

## 1) Role & quality bar

* Cursor must act as **Principal Mobile/Software Engineer**: enforce Clean Architecture boundaries, SRP, immutability, and 100% unit coverage for domain/usecases; ≥80% coverage overall.
* No business logic in widgets. UI reacts to BLoC **states** only. ([Bloc][5])

## 2) Clean Architecture contracts

* **Presentation** depends only on **domain** abstractions (`UseCase` interfaces), never on `dart:io` or plugins.
* **Domain** is pure Dart (no Flutter).
* **Data** binds to platform: DNS-SD, sockets, storage.
* Dependencies flow **inward** (interfaces in inner layers, implemented by outer layers). ([blog.cleancoder.com][6])

## 3) BLoC standards

* Each feature has its own Bloc/Cubit; **events in, states out**. No side-effects in builders; use `BlocListener` for navigation/snackbars. ([Bloc][5])
* States are **immutable** (e.g., `equatable` or `freezed`).
* Long-running ops (file send) expose **progress states** with percentage, speed, ETA.

## 4) Dependencies (chosen carefully)

| Purpose              | Package                              | Why                                                                            |
| -------------------- | ------------------------------------ | ------------------------------------------------------------------------------ |
| State mgmt           | `flutter_bloc`                       | battle-tested, predictable, great docs. ([Bloc][16])                           |
| Equality             | `equatable`                          | value equality for states/events (fewer rebuilds).                             |
| Discovery            | `multicast_dns` **or** `flutter_nsd` | mDNS in pure Dart or platform NSD/Bonjour. ([Dart packages][7])                |
| Transport (web opt.) | `flutter_webrtc`                     | encrypted RTCDataChannel across browsers/desktop/mobile. ([Dart packages][15]) |
| UUID                 | `uuid`                               | stable per-install identity.                                                   |
| Files                | `file_picker`, `path_provider`       | pick files & resolve safe app paths.                                           |
| DI                   | `get_it` (or provider)               | explicit dependency wiring.                                                    |

> Note: For **web**, never import `dart:io`; compile-time conditional imports keep the web target on WebRTC/WebSockets only. Browsers can’t use `dart:io`. ([Dart][4])

## 5) Code patterns

**Use case base:**

```dart
abstract interface class UseCase<Out, In> { Future<Out> call(In params); }
```

**ThreadBloc state slices:**

```dart
sealed class ThreadState {}
class ThreadIdle extends ThreadState {}
class ThreadLoading extends ThreadState {}
class ThreadLoaded extends ThreadState { final List<Message> messages; }
class TransferProgress extends ThreadState { final String id; final int sent; final int total; }
class ThreadError extends ThreadState { final String msg; }
```

**Repository interfaces (domain):**

```dart
abstract interface class DiscoveryRepository {
  Stream<Peer> browse();
  Future<void> advertise(Peer me);
  Future<void> stop();
}
abstract interface class TransportRepository {
  Future<void> listen(int port);
  Future<Connection> connect(Endpoint ep);
  Future<void> sendText(Connection c, String text);
  Future<void> sendFile(Connection c, Stream<List<int>> bytes, int size);
  Future<int> requestResumeOffset(Connection c, String transferId);
}
```

## 6) Testing strategy

* **Unit tests** (pure Dart):

  * `DiscoverPeers` emits peers from a fake discovery stream.
  * `SendFile` splits data into chunks; verify headers, sequence, CRCs.
  * `ResumeTransfer` seeks by `offset`.
* **Bloc tests**: use `bloc_test` to assert event→state transitions.
* **Widget tests**: Nearby list renders discovered peers; Thread shows progress + completion toast.
* **Integration** (desktop CI): loopback Socket tests with `ServerSocket`. ([Flutter API Docs][14])

## 7) Performance & resilience rules

* Backpressure: pause reading when sink’s `addStream` backpressure signals.
* Retries: exponential backoff for connect; resume after network change.
* Large files: memory-bounded chunking; temp files; fsync on completion.

## 8) Coding conventions

* Null-safety on; exhaustive `switch` on sealed classes.
* No plugin calls in `Bloc` constructors; inject repos.
* Lints: `pedantic`/`flutter_lints`; forbid `print` in production.

---

# README_POLICIES.md — Store policies, permissions, and common blockers

## 1) iOS / iPadOS

**Local Network privacy (iOS 14+)**

* Add **`NSLocalNetworkUsageDescription`** to Info.plist with a clear, human explanation (why you need LAN access). The system **prompts users** at first access. ([Apple Developer][10])
* Declare the exact **Bonjour service types** you browse/advertise in **`NSBonjourServices`** (e.g., `_lanshare._tcp`). Missing this can break discovery in release/TestFlight. ([Apple Developer][17])
* If you truly need **multicast/broadcast** beyond DNS-SD basics, you may need the entitlement **`com.apple.developer.networking.multicast`** (request approval in the Developer portal). Prefer DNS-SD to avoid this. ([Apple Developer][11])

**App Store Review**

* Follow Apple’s **App Review Guidelines**; ensure the app **still works** when Local Network is denied (e.g., show instructions to enable later), and explain the permission in-app **before** triggering the system dialog. ([Apple Developer][18])
* Multicast is discouraged for bulk transfer; use unicast/TCP for payloads. ([Apple Developer][19])

## 2) Android

**Permissions (Android 12/13+)**

* Use **`NEARBY_WIFI_DEVICES`** runtime permission if you manage or scan nearby Wi-Fi devices/APIs; set `neverForLocation` if you **don’t** derive location. Avoid legacy `ACCESS_FINE_LOCATION`. ([Android Developers][12])
* Use **NSD** to discover services on the LAN. ([Android Developers][8])

**Google Play requirements**

* Complete the **Data safety** form truthfully (even if you collect/share no data) and provide a valid **Privacy Policy URL** on the listing. Mismatches commonly cause rejections. ([Google Help][20])

## 3) Windows/macOS/Linux (desktop)

* **Windows**: the first listener bind will likely trigger a **Firewall allow** prompt. Document that users must allow the app on **Private** networks. ([Microsoft Support][13])
* **macOS App Store**: if sandboxed, add **network client/server** entitlements (`com.apple.security.network.client` / `...server`) when you listen for incoming TCP. (Apple’s entitlements overview applies.) ([Apple Developer][21])

## 4) Web build

* Browsers **cannot use `dart:io`**; therefore **TCP/UDP** sockets are blocked. Use **WebRTC DataChannels** (encrypted with DTLS) or WebSockets to a helper. Don’t ship a web build that tries to import `dart:io`. ([Dart][4])

## 5) Common blockers (and how to avoid them)

* **Forgotten iOS keys** → Add `NSLocalNetworkUsageDescription` and `NSBonjourServices` or you’ll fail discovery or get surprise prompts. ([Apple Developer][10])
* **Using UDP broadcast** on iOS without entitlement → traffic silently fails; stick to DNS-SD or request multicast entitlement. ([Apple Developer][11])
* **Android permission confusion** → for Wi-Fi APIs, request `NEARBY_WIFI_DEVICES` on API 33+ with `neverForLocation`. ([Android Developers][22])
* **Windows firewall** silently blocking inbound listeners → provide first-run help + link to allow the app on Private networks. ([Microsoft Support][13])
* **Web build uses `dart:io`** → won’t compile/run. Branch your transport to WebRTC for web. ([Dart][4])
* **Data safety mismatch** on Google Play → fill the form precisely and keep privacy policy consistent. (Research shows many apps misreport — avoid that.) ([Google Help][20])

## 6) Store listing & privacy policy checklist

* State clearly: **no internet servers**, transfers remain **on the local network**.
* Explain permissions:

  * iOS Local Network: needed to discover/send to peers on your Wi-Fi. ([Apple Developer][10])
  * Android Nearby Wi-Fi Devices: used solely for local discovery; not used for location. ([Android Developers][12])
* Provide a **privacy policy URL** and ensure it matches your Data Safety answers. ([Google Help][23])

---

## quick references

* **Dart sockets**: `Socket` / `ServerSocket` APIs. ([Dart API][9])
* **DNS-SD packages**: `multicast_dns`, `flutter_nsd`. ([Dart packages][7])
* **WebRTC DataChannels** (web): concepts & security. ([MDN Web Docs][3])
* **Android NSD docs**: how to browse/resolve services. ([Android Developers][8])
* **iOS Local Network privacy** & Bonjour keys. ([Apple Developer][10])

---

if you want, i can also generate **starter code scaffolds** for the packages (interfaces, example blocs, tests) so you can paste them into Cursor and hit run.

[1]: https://developer.apple.com/bonjour/?utm_source=chatgpt.com "Bonjour"
[2]: https://api.dart.dev/dart-io/?utm_source=chatgpt.com "dart:io library"
[3]: https://developer.mozilla.org/en-US/docs/Web/API/WebRTC_API/Using_data_channels?utm_source=chatgpt.com "Using WebRTC data channels - Web APIs | MDN - Mozilla"
[4]: https://dart.dev/libraries/dart-io?utm_source=chatgpt.com "dart:io"
[5]: https://bloclibrary.dev/flutter-bloc-concepts/?utm_source=chatgpt.com "Flutter Bloc Concepts"
[6]: https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html?utm_source=chatgpt.com "Clean Architecture - The Clean Code Blog - Uncle Bob"
[7]: https://pub.dev/packages/multicast_dns?utm_source=chatgpt.com "multicast_dns | Dart package"
[8]: https://developer.android.com/develop/connectivity/wifi/use-nsd?utm_source=chatgpt.com "Use network service discovery | Connectivity"
[9]: https://api.dart.dev/dart-io/Socket-class.html?utm_source=chatgpt.com "Socket class - dart:io library"
[10]: https://developer.apple.com/documentation/technotes/tn3179-understanding-local-network-privacy?utm_source=chatgpt.com "TN3179: Understanding local network privacy"
[11]: https://developer.apple.com/documentation/bundleresources/entitlements/com.apple.developer.networking.multicast?utm_source=chatgpt.com "com.apple.developer.networking.multicast"
[12]: https://developer.android.com/develop/connectivity/wifi/wifi-permissions?utm_source=chatgpt.com "Request permission to access nearby Wi-Fi devices"
[13]: https://support.microsoft.com/en-us/windows/firewall-and-network-protection-in-the-windows-security-app-ec0844f7-aebd-0583-67fe-601ecf5d774f?utm_source=chatgpt.com "Firewall and Network Protection in the Windows Security App"
[14]: https://api.flutter.dev/flutter/dart-io/ServerSocket-class.html?utm_source=chatgpt.com "ServerSocket class - dart:io library"
[15]: https://pub.dev/packages/flutter_webrtc?utm_source=chatgpt.com "flutter_webrtc | Flutter package"
[16]: https://bloclibrary.dev/?utm_source=chatgpt.com "Bloc State Management Library | Bloc"
[17]: https://developer.apple.com/documentation/bundleresources/information-property-list/nsbonjourservices?utm_source=chatgpt.com "NSBonjourServices | Apple Developer Documentation"
[18]: https://developer.apple.com/app-store/review/guidelines/?utm_source=chatgpt.com "App Review Guidelines"
[19]: https://developer.apple.com/news/?id=0oi77447&utm_source=chatgpt.com "How to use multicast networking in your app - Discover"
[20]: https://support.google.com/googleplay/android-developer/answer/10787469?hl=en&utm_source=chatgpt.com "Provide information for Google Play's Data safety section"
[21]: https://developer.apple.com/documentation/bundleresources/entitlements?changes=latest_min_2_3__8&utm_source=chatgpt.com "Entitlements | Apple Developer Documentation"
[22]: https://developer.android.com/about/versions/13/behavior-changes-13?utm_source=chatgpt.com "Behavior changes: Apps targeting Android 13 or higher"
[23]: https://support.google.com/googleplay/android-developer/answer/9859455?hl=en&utm_source=chatgpt.com "Prepare your app for review - Play Console Help"
