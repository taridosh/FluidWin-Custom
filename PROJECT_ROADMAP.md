# FluidWin Custom — Project Roadmap

> Status: planning / technical discovery. No application source code has been changed.
> Working branch: `docs/technical-audit-roadmap`
> Baseline: `main` at `8f695db5c46d1d4ecc08befb4ee2637d737314b6` (FluidWin 1.2.3)

## Product vision

Build a personal, highly configurable Windows desktop shell experience on top of FluidWin, preserving its existing visual language, acrylic/glass surfaces, animation quality, Dock, and Dynamic Notch. Extend its current app launcher into a real user-organized app hub and evolve the current weather/news panel into a reusable widget platform.

The Windows taskbar replacement and FluidWin's overall look are foundational product decisions. Booki is a source of interaction/design inspiration (especially grouped Dock popovers), not a target codebase to merge wholesale.

## Product principles

1. **Preserve FluidWin's identity.** New surfaces should look and behave like FluidWin, not like a second application bolted onto it.
2. **Extend before replacing.** Reuse the current Win32, Direct2D, DirectComposition, Direct3D and Shell integration wherever practical.
3. **One data source, multiple surfaces.** A widget/data provider should be reusable by the Dock, launcher, control center and Notch.
4. **User-controlled and reversible.** User layouts/settings must be persistent, validated, backed up/migratable and easy to reset.
5. **Keep the UI thread responsive.** Network, shell enumeration, sensor polling and expensive image work must run off the rendering/message thread.
6. **Incremental delivery.** Every phase should build, run and be testable before the next begins.
7. **No destructive changes to upstream baseline.** Work on feature branches; keep `main` as a known-good baseline and merge via reviewed PRs.

## Target experience

### A. Launcher 2.0
- Keep the current launcher opening behavior and visual style.
- Dedicated **All Apps** view listing installed applications.
- User-created folders/groups.
- Pin/unpin favorites.
- Manual ordering and drag-and-drop between positions/folders.
- Context menu actions (open, pin/unpin, move to folder, rename display label, remove shortcut where safe).
- Persist layout and restore it after restart.
- Search across all apps and user groups; preserve keyboard navigation and accessibility.
- Handle duplicate display names, removed apps, updated package identities and broken shortcuts gracefully.

### B. Widget platform
- A common widget definition/configuration model, rather than one-off drawing/data code for every widget.
- Initial widget types: clock/date (Gregorian/Jalali), system monitor, weather, finance/FX/crypto, RSS/news, image/GIF.
- Per-instance settings: size, placement, appearance, data source, refresh interval and enabled state.
- Support local files and user-supplied URLs only through safe, explicit configuration.
- A provider/data layer should cache values and expose snapshots to multiple UI surfaces.
- Define resource limits, timeouts, error states and refresh backoff.

### C. Dock groups and popovers
- A Dock item can represent a group.
- Clicking it opens a compact FluidWin-style popover inspired by Booki.
- Group contents may include apps, files, shortcuts and supported widgets.
- Custom group name/icon, ordering, drag/drop editing and persistence.
- Keep the existing app-window tracking and minimize/restore behavior intact.

### D. Dynamic Notch 2.0
- Preserve existing time, volume, music, notifications, timer, battery/headset alerts and shelf.
- Add configurable live-information slots: hardware readings, selected finance pairs, RSS headlines and widget summaries.
- Add optional motion/transition variants and per-source display rules.
- Consume the same cached provider values as Dock/launcher; do not create duplicate polling loops.
- Ensure Notch content never blocks the current notification/music/timer priority rules.

### E. Settings and profiles
- Dedicated sections for launcher, groups, widgets, Dock, Notch, data sources, calendar and appearance.
- Per-widget configuration and refresh policies.
- Optional export/import of user layout and settings.
- Versioned configuration schema and migration path.
- Keep existing settings backward compatible.

## Proposed architecture

This is a target design, not a commitment to introduce a large framework before the first feature.

```text
Windows Shell / APIs / Network providers
              |
       Data & Discovery Layer
       - app catalog / shell identity
       - widget providers / cache
       - hardware telemetry
       - RSS / weather / finance
              |
       User Configuration Store
       - launcher layout
       - Dock groups
       - widget instances
       - Notch display rules
       - schema version / migration
              |
          Presentation
       - Launcher
       - Dock + group popovers
       - Control Center / widget panel
       - Dynamic Notch
```

### Architectural boundaries
- **App catalog:** discovers installed apps and resolves stable launch identities; separates discovered apps from user organization.
- **User layout model:** stores app IDs, folder/group IDs, ordering and pinned state; does not mutate the Windows installed-app inventory.
- **Widget registry:** describes available widget types and their configuration/refresh/render contracts.
- **Provider/cache layer:** owns network/system polling and last-known-good snapshots. UI surfaces subscribe/read snapshots.
- **Presentation surfaces:** draw the same model using existing FluidWin visual primitives.
- **Settings persistence:** migrate current registry-backed settings without breaking existing installs. A versioned JSON file may be considered for complex user layouts, but the storage choice must be validated before implementation; do not scatter ad-hoc registry values across modules.

## Delivery phases

### Phase 0 — Technical audit and baseline
- [x] Confirm repository, branch, baseline commit and build recipe.
- [x] Inventory source tree and identify major UI/system modules.
- [x] Inspect key launcher, Dock, widget, Notch, sensor, settings and startup paths.
- [ ] Finish detailed function-level review of all affected modules and dependencies.
- [ ] Build and run the unmodified baseline on the target Windows machine.
- [ ] Capture baseline screenshots, behavior notes, performance and diagnostics.
- [ ] Record compiler/runtime prerequisites and known limitations.

**Exit criteria:** reproducible baseline build/run and a reviewed technical design with known integration points and risks.

### Phase 1 — UX/design specification
- Define launcher states, navigation, empty/search states, context menus and drag/drop behavior.
- Define folder/group editing flows and keyboard interactions.
- Create visual mockups in FluidWin style before implementation.
- Agree on interaction details and acceptance criteria.

**Exit criteria:** approved launcher wireframes/specification; no code implementation yet.

### Phase 2 — Launcher 2.0 foundation
- Separate discovered app catalog from user-managed launcher layout.
- Add versioned persistence for pins, ordering and folders.
- Implement All Apps, favorites, folders and manual ordering.
- Implement drag/drop and context menus.
- Preserve current async icon loading, filtering, focus handling and animation behavior.
- Add migration/default layout for existing users.

**Exit criteria:** launcher layout survives restart; all installed apps remain discoverable; search, keyboard, mouse and high-DPI behavior pass tests.

### Phase 3 — Widget/data foundation
- Define widget type/instance/provider contracts.
- Implement scheduler, cache, timeout, cancellation and failure/backoff handling.
- Move existing weather/RSS behavior behind the new provider model.
- Add calendar provider and hardware telemetry provider.
- Add first widgets incrementally: clock/calendar, CPU/RAM/GPU, weather.

**Exit criteria:** at least two different surfaces can read the same provider snapshot without duplicate polling; UI remains responsive under network/API failure.

### Phase 4 — Dock groups and popovers
- Extend Dock item model to support group entries without disturbing normal app entries.
- Build compact popover using existing glass/rendering infrastructure.
- Add apps/files/widgets to groups, editing and persistence.

**Exit criteria:** grouped Dock items work across monitor/DPI changes and do not regress window tracking/minimize behavior.

### Phase 5 — Notch live information
- Add provider-backed display slots and user-selected sources.
- Define content priority, rotation/scrolling and animation rules.
- Reuse provider cache; enforce refresh and display limits.

**Exit criteria:** Notch can display selected live data while existing music, notifications, timer and alert behavior remains reliable.

### Phase 6 — Settings, import/export and polish
- Add configuration UI for all new systems.
- Add schema migration, backup/reset and import/export.
- Add localization and accessibility/keyboard refinements.
- Profile CPU/GPU/memory and reduce unnecessary refresh/render work.

**Exit criteria:** complete user configuration, safe upgrade path, stable performance and release checklist.

## Proposed first implementation milestone

After Phase 0 and the visual design are approved, implement Launcher 2.0 first. It is the clearest user-facing feature and can be developed while leaving the current widget/Notch architecture mostly intact.

Suggested branch sequence:
- `feature/launcher-layout-model`
- `feature/launcher-ui-v2`
- `feature/widget-provider-core`
- `feature/dock-groups`
- `feature/notch-live-data`

Use pull requests into `main`; do not develop directly on `main`.

## Cross-cutting quality gates

- Build with the repository's documented MSYS2 UCRT64 / MinGW-w64 GCC toolchain and C++20 settings.
- Test Windows 11, multiple DPI scales, multiple monitors, small screen heights and light/dark themes.
- Test no-network, slow-network, malformed feeds, missing apps and removed shortcuts.
- Test clean install, upgrade from existing registry settings, restart and abnormal termination.
- Check idle CPU, GPU, memory, thread count and network activity.
- Do not require administrator rights unless a feature has a clearly documented reason.
- Do not silently collect or transmit user data. Network providers and endpoints must be explicit and documented.

## Open technical decisions

1. Whether complex user layouts should live in a versioned JSON file under the user's profile, or in a structured registry subtree. Recommendation: evaluate a small versioned JSON store for user-authored layouts, while preserving existing registry-backed scalar settings.
2. Whether third-party widgets are in scope for the first release. Recommendation: first ship built-in widgets with a stable internal contract; only later consider a documented external plugin model.
3. Which finance/FX data sources are reliable and reachable from the user's environment. This needs provider-specific research and graceful fallback design.
4. Which telemetry APIs and libraries provide CPU/GPU temperatures reliably on the target hardware. Windows does not expose every sensor uniformly; GPU vendor APIs or a maintained hardware library may be required.
5. Whether Jalali support should use Windows locale/calendar APIs or an explicit calendar conversion library. Decide after a small proof-of-concept and test against known Persian calendar dates.

## Current status

Only documentation/planning has been started on `docs/technical-audit-roadmap`. No C++ source, build script, release, or user configuration has been modified.
