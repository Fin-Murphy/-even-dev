# CLAUDE.md — Even Realities G2 development reference

Drop this file into any project that builds software for the **Even Realities G2** smart glasses.
Compiled 2026-11-25 from a multi-source, adversarially verified research pass. Things move fast
(SDK versions, doc URLs, firmware); **when a fact here conflicts with the live docs or the SDK's
`.d.ts` files, the live source wins.**

Source tags used below:
- **[OFFICIAL]** — evenrealities.com, hub.evenrealities.com docs, support.evenrealities.com, even-realities GitHub org
- **[COMMUNITY]** — reverse-engineered / third-party; unofficial, may break on any firmware update
- **[UNVERIFIED]** — single source or conflicting sources; confirm on hardware before relying on it
- **[G1-ONLY]** — applies to the previous-gen G1; do NOT assume it works on G2

---

## 1. Decision guide: which path am I building on?

| Path | When to use | Language | Needs Even app? | Status |
|---|---|---|---|---|
| **Even Hub plugin** (default) | Consumer apps, prototypes, anything you want distributed via Even Hub | TS/JS web app (Vite/React/vanilla) | Yes — runs in a WebView inside the Even Realities phone app | [OFFICIAL], live since ~Mar–Apr 2026 |
| **Enterprise SDK** | Your own native/Flutter app talking directly to the glasses, no Even app/account | Flutter, native iOS, native Android | No | [OFFICIAL], launched 2026-11-10; via enterprise partnership; 3 free dev device licenses per dev account |
| **Raw BLE (community)** | PC/desktop control, research, things the SDKs don't expose | Python (Bleak), TS (Bun/Node + noble), Kotlin | No (phone must be disconnected) | [COMMUNITY] |
| **Even AI Agent Configuration** | Route "Hey, Even" voice queries to your own LLM backend | Any server exposing OpenAI-style chat-completions | Yes | [OFFICIAL-ish, from community writeups] — Settings → Even AI → Agent Configuration, only visible to Even Hub developers |

**Default assumption for a new project: Even Hub plugin** unless the project README says otherwise.

---

## 2. Hardware specs (G2)

| Item | Value | Tag |
|---|---|---|
| Launch | 2025-11-12, $599 (R1 ring $249) | [OFFICIAL] |
| Weight / build | 36 g, magnesium + titanium, IP65 | [OFFICIAL] |
| Display | Binocular waveguide, micro-LED, **monochrome green**, 60 Hz, 1200 nits, 98% passthrough, branded "HAO 2.0" | [OFFICIAL] |
| **App canvas** | **576 × 288 px per eye, 4-bit greyscale (16 levels of green), origin top-left** | [OFFICIAL] (Hub docs) |
| Physical panel | Support specs page says 640 × 350, 27.5° FoV — **apps cannot address this; use 576×288** | [UNVERIFIED] |
| Mics | 4-mic array → delivered to apps as a single 16 kHz PCM stream | [OFFICIAL] |
| Camera / speaker | **None.** No camera, no speaker. Use phone camera / phone audio. | [OFFICIAL] |
| IMU | Accelerometer/gyro, exposed via SDK `imuControl` (units & rates TBD in docs) | [OFFICIAL] |
| Input | Capacitive touchpads on both temples; optional **R1 ring** (same gestures) | [OFFICIAL] |
| Connectivity | Bluetooth 5.4 (hardware blog; Hub overview page still says 5.2), ~28 m range claimed | [OFFICIAL] |
| L/R arms | Linked by wired 0.1 mm FPC through the frame (G1 used wireless L↔R). **Each arm still advertises separately over BLE** (see §7) | [OFFICIAL] / [COMMUNITY] |
| Battery | Glasses 192 mAh (0.744 Wh), ~2 days typical; case 2000 mAh (7.4 Wh), ~7 recharges, contact charging, case is USB-C | [OFFICIAL] |
| Prescription | −12.00 to +12.00 | [OFFICIAL] |
| Offline | **Not supported** — all functions need BT to phone + internet | [OFFICIAL] FAQ |

**G1 → G2 deltas:** 1000→1200 nits, 2→4 mics, 20→60 Hz, ~75% larger display area, 1.5→2 day battery, BT 5.2→5.4, wireless L↔R → wired FPC, raw G1 opcodes → protobuf-over-custom-framing.

---

## 3. Built-in features (explore these first when the glasses arrive)

Dashboard (widgets), Even AI ("Hey, Even"), Conversate (live transcription + AI summary in app),
Prep Notes, AI Cues, Translate (35 languages), Teleprompt (voice-paced; handles simple text PDFs),
QuickNote / QuickList, Navigate (Google Maps data), Notifications (app shows in list only after it
has posted a phone notification), Calendar, Silent Mode, Even Hub (app store).

**Gestures (built-in UI):**
- Open Menu: firmware ≥ 2.2.9 → **tap then long-press 1 s**; older → long-press 1 s
- Swipe forward/back on the touchpad to navigate; tap to select
- Restart glasses: tap both touch panels 5 times [COMMUNITY]
- Even AI can be invoked while Navigate/Teleprompt are running and can voice-control Translate/Navigate

**Day-one setup checklist:**
1. Glasses ship in **shipping mode** (battery isolated) → put both arms in the case and close it to wake.
2. Install the **Even Realities** app (iOS: apps.apple.com/us/app/even-realities/id6747017725; also Android).
3. Pair, then **take the firmware update immediately** — most "won't connect" reports are old firmware.
4. Pair R1 separately: Devices → Add device → Even R1.
5. Enable dev access (§4.2).

---

## 4. Even Hub plugin development (primary path)

### 4.1 Architecture
- Your plugin is a **web app** loaded in a WebView inside the Even Realities phone app. **Code runs on the phone; glasses are display + input.**
- Sandboxed; permissions declared (e.g. `g2-microphone`, `phone-microphone`; camera/album with user consent).
- **Network:** all outbound requests must be whitelisted in `app.json` and pass CORS checks (docs: `/docs/build/networking`).
- Only **plugins** are live. Planned (not yet available): dashboard widgets, dashboard layouts, AI skills.
- **No built-in STT exposed** — you get raw PCM only; run your own speech-to-text (cloud or local Whisper etc.).
- Distributed as a single **`.ehpk`** package uploaded via the developer portal; apps can be private or beta to specific users.

### 4.2 Tooling & setup
```bash
# Node 20 LTS or 22+
npm install -g @evenrealities/evenhub-cli @evenrealities/evenhub-simulator
npm install @evenrealities/even_hub_sdk@latest
```
- **Dev access:** there is **no Developer Mode toggle**. Sign in at `hub.evenrealities.com/login` with the same account as the phone app → **force-quit** the phone app (swipe out of recents) → reopen. A developer section with **Scan QR** appears in the Even Hub tab.
- **Sideload to hardware:** run your dev server (Vite default `:5173`), then
  `evenhub qr --url "http://<your-lan-ip>:5173"` and scan with the app. Phone + laptop on same network.
- **Simulator:** `@evenrealities/evenhub-simulator` (≥ 0.9.0 needed to simulate long-press). Preview at `localhost:5173`.
- **Templates:** `github.com/even-realities/evenhub-templates` (minimal, ASR, image, text-heavy).
- **All-in-one kit:** `github.com/even-realities/everything-evenhub` (SDK, CLI, simulator, docs).
- **Authoritative API reference = the SDK's shipped `.d.ts` files.** Read `node_modules/@evenrealities/even_hub_sdk/**/*.d.ts` before guessing a signature.

**Sideload network gotchas:**
- macOS firewall silently drops the first inbound phone connection → allow the Node binary (System Settings → Network → Firewall → Options).
- Windows Defender blocks `node.exe`.
- Wi-Fi AP/client isolation (corporate, some home routers) silently breaks it → use phone hotspot or Tailscale.
- In-app "Link" jumps may fail (no response / wrong page / bounce home) → re-scan the QR instead.

### 4.3 Display / container model
| Limit | Value |
|---|---|
| Canvas | 576 × 288 px, 4-bit greyscale, origin top-left |
| Containers per page | **≤ 12 total: ≤ 8 text/list + ≤ 4 image** |
| Image container size | 20–288 px wide × 20–144 px tall (max 288×144) |
| Text per container | ≤ 1,000 chars in `createStartUpPageContainer` / `rebuildPageContainer`; ≤ 2,000 in `textContainerUpgrade` (wire level: 1,000 UTF-8 bytes — stay under 1,000 to be safe) |
| Event capture | **Exactly one** container per page has `isEventCapture: 1`; it receives all input |
| Text brightness | Per-container, levels 0–4 |
| Icons | 24 × 24 recommended; only some Unicode renders (see `/docs/build/design-guidelines`) |

Key calls: `createStartUpPageContainer` (must run first — also a prerequisite for glasses audio), `rebuildPageContainer`, `textContainerUpgrade`, `updateImageRawData`.

**Display gotchas:**
- **The simulator allows ~6 image containers; hardware allows 4.** The simulator will hide layout failures.
- **BLE bandwidth is narrow.** Frequent image updates back up, freeze rendering or drop the connection. The simulator does NOT reproduce this. Prefer text; throttle/debounce image updates; test on hardware.
- Downscale images (phone camera/album returns base64) before `updateImageRawData`.
- Monochrome green, transparent display: black = transparent. Design for high contrast, little text, glanceable.

### 4.4 Input events
```ts
import { waitForEvenAppBridge /*, OsEventTypeList, AudioInputSource, ImuReportPace */ } from '@evenrealities/even_hub_sdk';

const bridge = await waitForEvenAppBridge();
bridge.onEvenHubEvent((event) => {
  // text container with capture -> event.textEvent
  // list container with capture -> event.listEvent
  // long press (any)            -> event.sysEvent (no eventSource)
  // contextual menu selection   -> event.menuItemClickEvent
  // IMU sample                  -> event.sysEvent, eventType IMU_DATA_REPORT, x/y/z floats
  // audio                       -> event.audioEvent.audioPcm (Uint8Array)
});
```
| OsEventTypeList | Value | Meaning |
|---|---|---|
| `CLICK_EVENT` | 0 | Single press (G2 or R1) |
| `SCROLL_TOP_EVENT` | 1 | Swipe up |
| `SCROLL_BOTTOM_EVENT` | 2 | Swipe down |
| `DOUBLE_CLICK_EVENT` | 3 | Double press |
| `LONG_PRESS_EVENT` | 9 | Long press begins (SDK ≥ 0.0.14, Even App ≥ 2.2.9) |
| `LONG_PRESS_RELEASE_EVENT` | 10 | Long press released (same version gate) |

**Input gotchas:**
- **Click can arrive as `undefined`** (SDK normalizes 0 → undefined). Always handle `case undefined:` as click.
- Tap-then-long-press is **reserved by the OS** for the system menu.
- Event source field distinguishes temple vs R1 (except long press, which has none).
- Older SDKs/blog posts say "no long press" — that was true before SDK 0.0.14.

### 4.5 Audio
```ts
await bridge.audioControl(true, AudioInputSource.Glasses); // default; or AudioInputSource.Phone
// ... event.audioEvent.audioPcm: Uint8Array, event.audioEvent.source
await bridge.audioControl(false);
```
- Format (both sources): **PCM, 16 kHz, signed 16-bit little-endian, mono.**
- Glasses mic requires `createStartUpPageContainer` to have run first + `g2-microphone` permission; phone mic needs `phone-microphone`.
- No output audio on glasses — play sound on the phone if needed.

### 4.6 Other device APIs
- **IMU:** `bridge.imuControl(isOpen, ImuReportPace.P100 … P1000)` — pace values are protocol codes, **not Hz**. Units TBD in docs.
- **Phone camera/album:** `captureImageFromCamera()`, `pickImageFromAlbum()` → single `AppImageAsset` (base64).
- **Location (phone GPS):** `getAppLocation`, `startAppLocationUpdates`, `onAppLocationChanged`.
- **Device:** `getDeviceInfo()` → model (G1/G2/Ring1), serial, battery, wearing, charging, in-case; `onDeviceStatusChanged`.
- **User / storage:** `getUserInfo`, `setLocalStorage` / `getLocalStorage`.

---

## 5. Enterprise SDK (native / Flutter)
- Announced 2026-11-10: `evenrealities.com/blogs/news/introducing-the-even-g2-enterprise-sdk`
- Repos: `github.com/even-realities/enterprise_sdk` (iOS/Android), `github.com/even-realities/flutter_enterprise_sdk` [UNVERIFIED URLs — from secondary index]
- Runs inside your app; talks BLE directly to glasses; **no Even server in runtime path**, license verified locally on glasses; end users don't need Even app/account.
- Same component model as Even Hub: text, lists, images, menus, input events, mic audio, motion data.
- 3 free device licenses per registered dev account; production pricing via `enterprise.evenrealities.com`.
- Apache-2.0 source release is *planned*, not done.

---

## 6. Raw BLE protocol (G2) — [COMMUNITY], unofficial

Use only when the SDKs can't do what you need. Tested on stock firmware ~v2.2.20; **any firmware update can break it.**

| Item | Value |
|---|---|
| Advertised names | `Even G2_XX_L_YYYYYY` (left), `Even G2_XX_R_YYYYYY` (right) |
| UUID base | `00002760-08c2-11e1-9073-0e8ac72e{xxxx}` |
| `…0000` | Main service |
| `…5401` | Command write (write without response) |
| `…5402` | Notify / responses (enable via CCCD `0x0100`) |
| `…6402` | Display rendering (write w/o response, 204-byte packets; semantics still under research) |
| MTU / interval | 512 bytes; conn interval ~7.5–30 ms |
| Pairing | **No standard BLE pairing/bonding.** Custom 7-packet app-level handshake (capability queries + time sync w/ timestamp + transaction_id) |
| Packet | `[AA][21][seq][len][01][01][svc_hi][svc_lo][payload…][crc_lo][crc_hi]` (8-byte header) |
| CRC | CRC-16/CCITT-FALSE (init 0xFFFF, poly 0x1021) over **payload only**, little-endian |
| Payload | Protobuf (see `i-soxi/even-g2-protocol/proto/`) |
| Service IDs (i-soxi) | 0x80 auth, 0x04 display wake, 0x06 teleprompter, 0x07 dashboard, 0x0B Conversate, 0x0E display config, 0x20 commit |
| Even Hub containers | Service 0xE0 (`sendPb(0xe0, …)`) — g2-kit only [UNVERIFIED] |
| Mic audio | LC3-compressed; decode with `google/liblc3` → 16 kHz s16le mono |
| Working timings (gateway) | 8 ms inter-packet gap, 5 s heartbeat, 200 ms between image fragments |

Status per i-soxi: teleprompter/custom text ✅, calendar widget ✅, notifications partial (metadata), navigation / Even AI / translate / 0x6402 rendering = research.

**To connect from a PC:** disconnect the phone first (turn phone BT off and force-close the Even/MentraOS app; no need to unpair) — glasses won't advertise while connected. g2-kit notes pairings can be lost when glasses sleep.

---

## 7. Repos & libraries

**Official (github.com/even-realities):**
- `everything-evenhub` — SDK + CLI + simulator + docs bundle
- `evenhub-templates` — 4 starter apps
- `EvenDemoApp` — Flutter, BSD-2 — **[G1-ONLY]**, background reading only
- `EH-InNovel`, `lvgl-sys-v9` — misc

**Community (G2):**
- `github.com/pangoleen/awesome-even-realities-g2` — **curated index; check this first for new tools**
- `github.com/i-soxi/even-g2-protocol` — protocol docs, `.proto` files, Python teleprompter example
- `github.com/Commute773/g2-kit` — MIT, TypeScript, Bun (Node 22+), `@stoprocent/noble` peer dep, protobuf builders + UI abstractions, LC3 decoder; read `ble/docs/gotchas.md`
- `github.com/gpsnmeajp/men-g2-ble-gateway` — Python 3.11+/Bleak, BLE ↔ HTTP/WebSocket gateway, port of MentraOS `G2.kt`; Zenn writeup: `zenn.dev/gpsnmeajp/articles/dbf9846091f493`
- MentraOS — open-source smart-glasses OS with a G2 driver (`G2.kt`)
- Also referenced in the awesome list: `even-g2-notes` (architecture/input/packaging notes), OpenEvenSdk (Python/Swift/Kotlin bridges), evenRealities-openCFW, Even-G2-RE

**[G1-ONLY] — do not use for G2 without porting:** `even_glasses` (PyPI, GPL-3.0, emingenc), Gadgetbridge G1 support, EvenDemoApp opcodes (0x4E text, 0x15 BMP, 0xF5 touch, 0x0E mic, 0xF1 audio, 576×136 1-bit BMP).

---

## 8. Docs & support URLs

- Dev docs: `https://hub.evenrealities.com/docs` — key pages:
  - `/docs/get-started/overview`, `/docs/get-started/quickstart/install-tools`, `/docs/get-started/quickstart/hardware`
  - `/docs/build/display`, `/docs/build/device-apis`, `/docs/build/networking`, `/docs/build/design-guidelines`
  - `/docs/reference/cli`
  - (old `/docs/getting-started/*` and `/docs/guides/*` paths redirect here)
- Dev portal / login: `https://hub.evenrealities.com/login` (publishing portal also referenced at `evenhub.evenrealities.com`)
- Developer Discord: linked from the Hub docs overview page
- Support (G2): `https://support.evenrealities.com/hc/en-us/categories/13489714076815-G2`
  - Specs `…/articles/13499229138959`, General FAQs `…/articles/13499250872335`, User Guide `…/articles/13492158323471`, Quick Guide PDF `…/articles/13754865887375`
  - Connection troubleshooting: `13499349578255`, `14834079818127`, `17123887466255`; blank lenses `17135065037839`
- Engineering blog (G1→G2): `evenrealities.com/en-CA/blogs/even-insider/how-we-rebuilt-g2-from-the-inside-out`
- Platform philosophy: `evenrealities.com/en-IE/blogs/even-insider/the-unfinished-glasses`
- Practitioner dev tips: `dev.to/yokomachi/got-an-even-g2-build-your-own-apps-hands-on-impressions-and-dev-tips-1glp`

---

## 9. Rules for working in a G2 project

1. **Check the SDK `.d.ts` before writing any bridge call.** Do not invent method names or enum values.
2. **Design for 576×288 monochrome green, ≤12 containers (≤4 images), one event-capture container.** Validate limits in code, not just in the simulator.
3. **Text first, images sparingly.** Throttle display updates; treat BLE as a slow, lossy pipe.
4. **Handle `undefined` as click** in every event switch.
5. **Call `createStartUpPageContainer` before audio** and request the right permission.
6. **Whitelist every network origin in `app.json`;** expect CORS to apply.
7. **No offline, no speaker, no camera, no on-glasses STT** — plan the phone/cloud side accordingly.
8. **Test on hardware before calling a feature done.** Simulator ≠ device (image limits, bandwidth, timing).
9. **Tag any raw-BLE code as unofficial**, pin the firmware version it was tested on, and isolate it behind an interface.
10. **Never apply G1 protocol knowledge to G2** without confirming it.

## 10. Open questions (verify when hardware arrives)
- Exact image encoding/bit depth over the SDK and max sustainable update rate.
- IMU units and real sample rates per `ImuReportPace`.
- R1 ring specs, battery, pairing quirks; whether R1 health data is exposed to apps.
- Even Hub publishing: review process, terms, monetization.
- Whether Developer Mode / sideloading survives firmware updates.
- Enterprise SDK availability for individuals.
