# FluidWin Custom — Technical Audit

> Audit date: 2026-11-17
> Repository: `taridosh/FluidWin-Custom`
> Baseline branch/commit: `main` / `8f695db5c46d1d4ecc08befb4ee2637d737314b6`
> Upstream parent: `clooudfn/FluidWin`
> This is a first-pass architectural audit of the complete source inventory and the critical paths related to the requested features. Detailed line-by-line review and a local Windows build remain Phase 0 tasks.

## 1. Repository and build facts

- Public GitHub fork; default branch is `main`.
- Current baseline commit is FluidWin 1.2.3, authored upstream, with the fork pointing at the upstream release commit.
- Repository license: MIT.
- Source is primarily C++ and organized as a single application translation unit with many implementation headers included from `fluidwin.cpp`.
- Build script: `compila.ps1`.
- Documented toolchain: MSYS2 UCRT64, `mingw-w64-ucrt-x86_64-gcc`, GNU C++20, static link.
- Graphics stack: Win32 windows, Direct2D, DirectComposition, Direct3D 11 / DXGI, DirectWrite, DWM acrylic backdrop.
- Platform integrations include Windows Shell/known folders, registry, WinHTTP, WASAPI, WinRT radio APIs, power status, notifications and monitor APIs.
- The application is designed to run without administrator rights and restore Windows state when it exits.

## 2. Source inventory

| File | Responsibility / relevance |
|---|---|
| `fluidwin.cpp` | Main process, message loop, hooks, window lifecycle, animation coordination, startup/shutdown and integration glue. |
| `dock.h` | Dock model/rendering, app tracking, pinned taskbar shortcuts, system status, shared graphics resources, launcher state and control-panel coordination. |
| `launcher.h` | Launcher window, app enumeration, async icon loading, search/filtering, input and rendering. |
| `menu.h` | Custom acrylic context menus and shell menu adaptation; useful for launcher/Dock context actions. |
| `widget.h` | Current weather and RSS data acquisition, background thread, cache/snapshot and parsing. |
| `centro.h` | Control Center, widget/weather/news panel, notifications/calendar and other panels. |
| `notch.h` | Dynamic Notch rendering, animation/state machine, music, volume, notifications, timer, alerts and file shelf. |
| `sensori.h` | Microphone/webcam usage, battery and Bluetooth audio-device observations. |
| `impostazioni.h` | Settings model/UI and persistence to HKCU registry. |
| `testi.h` | Interface language and date locale selection (currently Italian/English). |
| `gpu.h` | Shared D3D11 device/context. |
| `sistema.h` | Wi-Fi/Bluetooth, theme, audio output and monitor brightness operations. |
| `ricerca.h`, `panoramica.h` | Search and Alt+Tab/window overview. |
| `musica.h`, `notifiche.h`, `banner.h`, `mixer.h`, `appunti.h` | Media session, notification capture/display, audio mixer and clipboard history. |
| `aggiorna.h` | Release update checking/downloading. |
| `affianca.h` | Window snapping behavior. |
| `vetro.h`, `raggio.h`, `fabbriche.h`, `scala.h`, `pool.h`, `viva.h` | Shared graphics, window, DPI, thread-pool and live-capture utilities. |
| `compila.ps1`, `icona.rc`, `icona.ico`, `icona.svg` | Build and application resources. |
| `docs/gif/*` | Existing UX references and interaction demonstrations. |

## 3. Existing architecture

### 3.1 Process and rendering model
- `fluidwin.cpp` owns startup/shutdown, message dispatch and global integration.
- `dock::Motore` in `dock.h` owns Dock state and creates the launcher, panels and Notch.
- The Dock and launcher use Direct2D/DirectComposition surfaces. `gpu.h` exposes one shared D3D11 device/context to avoid duplicating expensive graphics devices.
- The Notch has its own window and rendering path (DIB/software Direct2D render target), with its own scene/state machine.
- Background work is already used for app enumeration, icon extraction and weather/RSS downloads. Results are handed back to the UI thread via messages/snapshots.
- This architecture is capable of incremental extension, but shared data contracts need to be introduced deliberately; avoid moving rendering code into provider threads.

### 3.2 Launcher today
- The launcher is already a custom animated window, opened by the Windows key or Dock's App icon.
- Installed apps are enumerated through the Shell AppsFolder (`FOLDERID_AppsFolder`), converted to `shell:AppsFolder\\...` launch targets, filtered and sorted.
- Search supports prefix/word-prefix, initials and subsequence matching.
- Icons are loaded asynchronously; the app list is refreshed periodically and current selection/scroll is preserved when refresh results arrive.
- Existing UI is a fixed six-column grid with a search box, power menu and scrolling.
- App list is discovered from Windows, but user organization is not separated from the discovered catalog.
- No persistent user-managed favorites/folders/manual order/drag-drop layout model was found in the inspected launcher path.
- The current launcher is therefore a strong rendering/input foundation, but its data model needs a distinct user-layout layer.

**Implication:** keep the Shell catalog as the authoritative inventory; layer a persistent user model over it. Do not persist raw display names as the only app identity. Prefer launch identity plus a resilient fallback/resolution strategy for package updates and broken links.

### 3.3 Dock today
- Dock entries combine pinned Windows taskbar shortcuts and running applications.
- It tracks executable paths, AppUserModelIDs, Squirrel app version paths and packaged apps.
- It has custom app icons, launch/minimize/restore tracking, async icon loading, status area, downloads stack, recycle bin and Control Center.
- It reserves the taskbar area only when configured; it otherwise hides the Windows taskbar and restores the prior state on shutdown.
- It handles multiple monitors and DPI-aware positioning.
- Dock entries are currently built from system/taskbar state, not from a general-purpose user-authored group model.

**Implication:** introduce a separate user-configurable Dock item/group model while retaining the current runtime app/window tracker. A group must not be mistaken for an application window entry.

### 3.4 Widget data today
- `widget.h` currently contains a small domain-specific `Dati` service and an `Istantanea` snapshot for weather and news.
- Weather uses Open-Meteo via WinHTTP; city/coordinates are read from the registry.
- News uses the ANSA RSS feed.
- Refresh is limited to approximately once per 10 minutes and is requested when the widget panel opens.
- Network requests run on a background thread, use timeouts, check HTTP status and cap response size.
- JSON parsing is lightweight/manual and tailored to the known Open-Meteo response; RSS parsing is tailored to the feed format.
- `centro.h` renders the weather/news panel directly from the snapshot.

**Implication:** this is not yet a general widget framework. Extract provider contracts, typed snapshots, scheduling/cache/error state and per-instance configuration before adding many widget types. Keep the existing weather/RSS output working during migration.

### 3.5 Dynamic Notch today
- `notch::Notch` has a distinct state machine with scenes for clock, music, open/expanded, volume, notification, alert, timer, sound and shelf.
- It already handles animations, notification callbacks, music integration, drag/drop shelf, battery/headset/microphone/camera signals.
- Notch display is currently driven by direct methods/callbacks from the main app and sensor subsystem.
- The Notch is not currently a generic host for arbitrary widget instances or provider subscriptions.

**Implication:** add a small live-info presentation contract and priority scheduler, but do not replace the existing scene machine. Live items must yield to high-priority events such as notifications, volume and timer.

### 3.6 Sensors and hardware
- `sensori.h` currently covers mic sessions, webcam usage, battery state/charging and newly connected Bluetooth audio outputs/headset battery when Windows exposes it.
- `sistema.h` handles Wi-Fi/Bluetooth radio state, theme, audio endpoint selection and monitor brightness.
- `gpu.h` only creates/shares the D3D11 graphics device; it is not a GPU telemetry provider.
- No complete CPU/GPU/RAM/temperature telemetry framework was found in the inspected paths.

**Implication:** system monitoring is a separate provider project. CPU/RAM can use Windows performance/system APIs; GPU utilization/temperature and CPU package temperatures may require vendor APIs or a maintained hardware library. Temperature availability must be proven on the target PC rather than assumed.

### 3.7 Settings and persistence
- Existing settings are represented by `impostazioni::Valori` and stored under `HKCU\\Software\\FluidWin\\Impostazioni`.
- The settings UI is grouped into General, Dock, Animations, Notch/notifications, Keyboard/mouse and Cursor.
- Current scalar settings are loaded/saved through typed registry values; user-authored launcher layouts and widget instances are not represented in this settings model.
- Weather location has its own registry values.

**Implication:** preserve existing registry values for compatibility. For complex user-created collections, evaluate a versioned structured store (likely JSON in the user's profile) with atomic writes, validation, backup and migrations. Final choice should be made after checking the project's file/path conventions and failure recovery.

### 3.8 Jalali calendar finding
- `testi.h` currently maps the interface language to fixed locale names: Italian uses `it-IT`, English uses `en-US`.
- Dock date and calendar labels use `GetDateFormatEx` with that fixed locale.
- Calendar grid logic in `centro.h` uses Gregorian `SYSTEMTIME`, Gregorian month lengths and Gregorian leap-year rules.
- Therefore, selecting the Persian calendar in Windows does not automatically make FluidWin's date/calendar UI Jalali: its locale and month-grid calculations are explicitly Gregorian/Italian-or-English.

**Implication:** the calendar needs an explicit calendar selection/formatting design. A simple locale-string substitution may format labels but will not correctly convert the month grid. Prototype Windows calendar APIs or a dedicated Jalali conversion implementation and validate against trusted Persian calendar dates before touching the UI.

## 4. Integration points and recommended changes

| Feature | Existing integration point | Recommended approach |
|---|---|---|
| Launcher folders/favorites/order | `dock::Motore::tutteApp`, `filtrate`, `launcher.h` | Keep discovered app catalog immutable; add user layout model and view projection. |
| Launcher drag/drop | `launcher.h` input handlers and render grid | Add explicit drag state, drop targets and model-level reorder/move operations. |
| Context menus | `menu.h` and existing launcher shell context menu | Reuse menu renderer; add app/folder-specific commands. |
| Dock groups | `dock.h` `Voce` model and click handling | Introduce a typed Dock entry/group variant; preserve runtime app tracking separately. |
| Popovers | Dock glass/window/DirectComposition infrastructure | Reuse current glass and D2D primitives; avoid a separate web UI runtime. |
| Widget providers | `widget.h`, `centro.h` | Extract provider/cache layer; keep rendering in presentation modules. |
| Hardware widgets | `sensori.h`, new provider module | Extend existing sensor scope carefully; isolate vendor-specific dependencies. |
| Notch live data | `notch.h` scene machine and `dock.h` update loop | Pass immutable snapshots to Notch; add explicit priority/rotation policy. |
| Settings | `impostazioni.h` | Add sections and per-widget/group editor; preserve old registry settings. |
| Jalali | `testi.h`, `dock.h::aggiornaStato`, `centro.h` calendar renderer | Introduce calendar abstraction for date conversion, month length, weekday and labels. |

## 5. Main technical risks

1. **Single translation unit / header-heavy code:** adding broad frameworks can increase compile time, coupling and accidental ODR/name collisions. Keep modules cohesive and use clear namespaces/contracts.
2. **UI-thread latency:** shell enumeration, icon extraction, network requests and telemetry must remain asynchronous with bounded queues and cancellation.
3. **Graphics resource lifetime:** Direct2D/DirectComposition resources are device/context-bound. UI rendering must own and release resources on the appropriate thread.
4. **App identity instability:** Windows Store package paths can change with versions; shortcut targets can disappear. User layout needs resilient identity resolution.
5. **Configuration corruption/migration:** drag/drop edits and app upgrades must not lose the user's custom layout.
6. **External data reliability:** endpoints can be unreachable, rate-limited or return changed payloads. Cache last-known-good values and show stale/error state.
7. **Sensor availability:** temperature readings vary by hardware/vendor and cannot be promised universally.
8. **Notch event conflicts:** live data rotation must not interrupt notifications, media controls, timer or volume.
9. **Calendar correctness:** Jalali conversion and leap-year rules must be tested independently from visual formatting.
10. **Regression risk:** FluidWin changes Windows taskbar state, window hooks and system behavior. Each feature needs a clean shutdown/recovery test.

## 6. Baseline build and validation

The repository documents this build path:
1. Install MSYS2.
2. Install `mingw-w64-ucrt-x86_64-gcc` in UCRT64.
3. Run `powershell -ExecutionPolicy Bypass -File compila.ps1` (optionally pass `-msys`).
4. Confirm `FluidWin.exe` is produced and launches.

Not yet verified by this remote source audit:
- A local build on the user's PC.
- Runtime behavior on the user's exact Windows build.
- Performance/resource baselines.
- DPI/multi-monitor regressions.
- Jalali conversion proof-of-concept.

## 7. Audit conclusion

FluidWin already provides a strong native Windows foundation: a custom animated launcher, robust Shell integration, an advanced Dock, a rich Notch state machine, background work patterns, shared graphics resources and registry-backed settings.

The requested project should **not** start with a rewrite or a Booki code merge. The highest-leverage change is to add a user-layout model around the existing app catalog and evolve the existing weather/news data service into a provider/cache layer only after the Launcher 2.0 direction is approved.

Next action after this documentation: finish the remaining function-level review, build/run the baseline on Windows, then design the Launcher 2.0 screens and interactions before implementation.
