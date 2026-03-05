# SaneBar Feature Requests & Roadmap

> **Navigation**
> | Session | How to Work | Releases | Testimonials |
> |---------|-------------|----------|--------------|
> | [../SESSION_HANDOFF.md](../SESSION_HANDOFF.md) | [../DEVELOPMENT.md](../DEVELOPMENT.md) | [../CHANGELOG.md](../CHANGELOG.md) | [testimonials.md](testimonials.md) |

Tracking user-requested features from GitHub, Reddit, Discord, and support emails.
Priority based on: frequency of requests, alignment with vision, implementation effort.

**Last audited:** 2026-03-04 (v2.1.20) — all items verified against codebase.

---

## Open Feature Requests

Research completed 2026-03-04. Each feature has a verified implementation plan and breakage rating.

### 1. Gradient Tint (Two-Color)
**Priority: LOW** | **Requests: 0** | **Effort: ~1 day** | **Breakage Risk: 1/5**

Both Ice and Bartender offer gradient tints. SaneBar only has a flat color fill (line 473 of `MenuBarAppearanceService.swift`).

**Implementation (verified against codebase):**
- Add 6 fields to `MenuBarAppearanceSettings`: `useGradientTint: Bool`, `tintColorGradient`/`tintOpacityGradient` (light), `tintColorGradientDark`/`tintOpacityGradientDark` (dark), `gradientAngle: GradientAngle` enum
- Create one computed `tintFill: some ShapeStyle` property that returns `LinearGradient` or flat `Color`
- Both render paths (fallback tint + Liquid Glass overlay) consume the same property
- Settings UI: toggle + direction picker + second color/opacity row in Appearance settings
- ~135 lines of new code across 3 files
- Zero migration: all new fields use `decodeIfPresent` with defaults. Existing users get `useGradientTint = false` — no behavior change.

**What could break:**
- Liquid Glass + gradient overlay blending needs visual testing on macOS 26 (medium concern)
- Reduce Transparency mode: must apply `max(opacity, 0.5)` clamp to gradient end opacity too
- Rounded corners: gradient renders before `clipShape` — correctly cropped, verified in code

**Files:** `MenuBarAppearanceService.swift`, `AppearanceSettingsView.swift`, `PersistenceService.swift`

---

### 2. New Icon Placement Control
**Priority: MEDIUM** | **Requests: 0** | **Effort: 2-3 days** | **Breakage Risk: 3/5**

Bartender lets you choose whether newly installed apps go to hidden or visible zone automatically. Every new app install currently breaks the user's clean bar.

**Implementation (verified against codebase):**
- SaneBar has **no persistent icon registry today**. The closest is `alwaysHiddenPinnedItemIds`. Each icon has a stable `uniqueId` (bundle ID or `bundleId::statusItem:N`).
- **Detection:** Hook into existing `NSWorkspace.didLaunchApplicationNotification` observer in `MenuBarManager.setupObservers()` + a 30s periodic background scan for icons that appear without a launch notification (helper processes, etc.)
- **Moving:** Reuse the existing battle-tested `moveIconAndWait()` from `MenuBarManager+IconMoving.swift`. Must follow the **shield pattern** from `enforceAlwaysHiddenPinnedItems()`: call `showAll()` → wait 300ms → scan → move → `restoreFromShowAll()`.
- **Settings:** `NewIconPlacementPolicy` enum (`.doNothing`, `.moveToHidden`, `.moveToAlwaysHidden`), plus `knownMenuBarItemIds: Set<String>` for tracking what we've seen before.
- New file: `MenuBarManager+NewIconPlacement.swift`

**What could break (this is the risky one):**
- **Moving without the shield pattern** silently fails or corrupts positions — MUST use `showAll()` first
- **System icon filtering** — Clock, Control Center, and all `com.apple.*` must be excluded via `isUnmovableSystemItem`
- **Race with always-hidden pin enforcement** — must cancel `alwaysHiddenPinEnforcementTask` before starting (existing pattern in `moveIcon()`)
- **False positives** — transient icons, Focus Mode extras, macOS Shortcuts menu extras. Mitigated by 2s delay after launch notification and never re-triggering for known `uniqueId`s
- **Position pre-seeding** — orthogonal, no conflict (pre-seeding only affects SaneBar's own items)
- **First run:** must populate `knownMenuBarItemIds` from a fresh scan on first enable, otherwise every existing icon looks "new"

**Files:** `PersistenceService.swift`, `MenuBarManager.swift`, new `MenuBarManager+NewIconPlacement.swift`, settings UI

---

### 3. Auto-Hide App Menus on Overlap
**Priority: MEDIUM** | **Requests: 1** | **Effort: 3-5 days** | **Breakage Risk: 2/5** | **GitHub: [#103](https://github.com/sane-apps/SaneBar/issues/103)**

| Requester | Request | Date |
|-----------|---------|------|
| btthle (GitHub) | "SaneBar refuses to open to show all menu bar icons... Ice has a setting that automatically hides the app menus" | Mar 2026 |

**How Ice does it (verified from Ice source code):**

*Detection:* Uses raw AX API to measure the app menu area:
```
AXUIElementCopyElementAtPosition(systemWide, displayOrigin.x, displayOrigin.y)
→ get kAXChildrenAttribute → filter to enabled items → union their frames
```
Returns a rect like `(10, 0, 348, 33)` covering File/Edit/View/etc. Compare against leftmost visible status item position.

*Hiding:* **No private APIs.** Ice exploits that macOS only shows the app menu for the frontmost app:
```
// Hide: steal focus → target app's menus vanish automatically
NSRunningApplication.current.activate(from: frontApp)
NSApp.setActivationPolicy(.regular)

// Restore: yield focus back → app menus reappear
NSApp.yieldActivation(to: targetApp)
NSApp.setActivationPolicy(.accessory)
```

**What SaneBar already has:**
- Leftmost visible icon X position known via `lastKnownSeparatorRightEdgeX`
- AX permission already granted, monitoring wired up
- `SaneActivationPolicy` in SaneUI already handles `.regular`/`.accessory` dance

**What could break:**
- **CRITICAL:** If `settings.showDockIcon = true`, calling `setActivationPolicy(.accessory)` during restore would unexpectedly hide the Dock icon. Must guard against this.
- `activate(from:)` and `yieldActivation(to:)` require macOS 14+. Need fallback for macOS 13 if supported.
- Brief focus steal is visible — the app menu items will flash/disappear. This is inherent to the technique.
- Some apps (Electron-based, Java) may not restore their menus cleanly after focus is yielded back.

**All APIs are public.** No App Store or notarization implications. No new entitlements.

**Files:** New `Core/Services/AppMenuHidingService.swift`, `MenuBarManager.swift` (observer hookup), `PersistenceService.swift` (setting), settings UI

---

### 4. Per-Profile Trigger Assignment
**Priority: LOW** | **Requests: 0** | **Effort: 3-5 days** | **Breakage Risk: 3/5**

SaneBar has profiles and triggers but they're disconnected. Bartender's triggers can activate a specific profile ("Join Work WiFi → switch to Work profile").

**Implementation (verified against codebase):**
- Every trigger service holds a weak `menuBarManager` and calls `manager.showHiddenItems()` when it fires. That one-liner becomes: `if let profileId = ... { manager.activateProfile(id:) } else { manager.showHiddenItems() }`
- `activateProfile` does a full `SaneBarSettings` replacement via `menuBarManager.settings = profile.settings`, which reactively restarts all 12+ dependent services through the existing Combine `$settings` observer.
- Add 6 fields to settings: `batteryTriggerProfileId: UUID?`, `appLaunchTriggerProfileId: UUID?`, `networkTriggerProfileId: UUID?`, `focusModeTriggerProfileId: UUID?`, `scheduleTriggerProfileId: UUID?`, `scriptTriggerProfileId: UUID?`
- UI: profile picker dropdown per trigger type in `RulesSettingsView`

**What could break (several real risks):**
- **State wipe:** A profile is a full settings snapshot from when it was saved. If the user adds new triggers AFTER saving the profile, loading it via trigger wipes those new triggers. Fix: when loading via trigger (automated), preserve all trigger-related fields across the swap. Manual loads from UI keep the existing full-replace behavior.
- **Circular trigger loops:** Work WiFi → load Work Profile → Work Profile has WiFi trigger → fires again → infinite loop. Fix: 5-second cooldown on `activateProfile`.
- **Auth bypass:** Trigger must not silently load a profile that disables `requireAuthToShowHiddenIcons`. Guard: if auth is currently ON and profile would turn it OFF, fall back to `showHiddenItems()`.
- **Deleted profile:** Profile assigned to trigger but user deletes it. `loadProfile` throws `profileNotFound`. Degrade to `showHiddenItems()`.

**Files:** `PersistenceService.swift`, new `MenuBarManager+Profiles.swift`, `NetworkTriggerService.swift`, `TriggerService.swift`, `FocusModeService.swift`, `ScheduleTriggerService.swift`, `ScriptTriggerService.swift`, `RulesSettingsView.swift`

---

## Open Bugs / Investigations

### Menu Bar Tint + Reduce Transparency (GitHub #20)
**Priority: LOW** | **Status: Deferred**

Tint overlay doesn't render when macOS "Reduce Transparency" is enabled. Reported on M4 Macs. Ice's tint works on same hardware.

**Research (Jan 2026):** SaneBar uses SwiftUI compositor blending; Ice uses AppKit `NSView.draw()` with Core Graphics. Unclear why one works and the other doesn't.

**Decision:** Deferred. No new reports. Revisit if more users hit this.

---

### Icon Positions Reset on Update (GitHub #92)
**Priority: HIGH** | **Status: Root cause found** | **Breakage Risk: 1/5**

User `flowsworld` reports icon positions reset when updating SaneBar while MacBook is disconnected from external monitor. 3+ occurrences (v2.1.12, v2.1.14, v2.1.18→2.1.20).

**Root cause (verified in code):** This is SaneBar's own bug, NOT macOS behavior. `positionsNeedDisplayReset()` in `StatusBarController.swift` actively wipes positions when screen width changes by >10%. The sequence:
1. User runs on external monitor (2560px). Width stored in `SaneBar_CalibratedScreenWidth`.
2. Sparkle updates, relaunches on laptop-only (1440px). Width delta = 43% → exceeds 10% threshold.
3. `resetPositionsToOrdinals()` wipes separator to ordinal 1 (right edge). Hidden zone gone.
4. User reconnects monitor. Ordinal positions stay. All icons now visible.

**No other app has this problem.** Ice, Bartender, Dozer all trust macOS persistence — macOS never deletes `NSStatusItem Preferred Position` keys on display change (only on `removeStatusItem()` or `isVisible = false`).

**Fix: Per-display position backup (~30-40 lines in `StatusBarController.swift`)**
- Before reset fires, save current pixel positions keyed by screen width: `SaneBar_Position_Backup_{width}_main`, `SaneBar_Position_Backup_{width}_separator`
- On return to a previously-seen display width, restore from backup instead of resetting to ordinals
- If no backup exists for the new width, behavior unchanged (ordinal re-seed)
- Backward compatible. No migration needed.

---

### Website Documentation Out of Date
**Priority: MEDIUM** | **Status: Open**

sanebar.com doesn't document many current features (notch-awareness, Browse Icons, visual zones, triggers, second menu bar, etc.). User `maddada_` (Discord) couldn't find docs for notch feature.

---

## Shipped Features (Archive)

All verified in codebase as of v2.1.20.

| Feature | Shipped | Tier | Requested By |
|---------|---------|------|-------------|
| Menu Bar Spacing Control | v1.0.14 (Jan 2026) | Pro | u/MaxGaav, u/Mstormer |
| Find Icon Speed Improvement | v1.0.12 | Free | u/Elegant_Mobile4311, bleducnx |
| Find Icon in Right-Click Menu | v1.0.x | Pro | u/a_tsygankov |
| Custom Dividers / Visual Zones | v1.0.x | Pro (extras) | u/MaxGaav |
| Secondary Menu Bar | v2.x | Pro | u/MaxGaav, bleducnx |
| Icon Groups | v2.x | Pro | macenerd (Reddit) |
| Visual Icon Grid | v1.0.x | Free | bleducnx |
| Auto-Disable on External Monitors | v1.0.15 (Jan 2026) | Pro | u/genius1soum |
| Scroll/Click Toggle | v1.0.x | Pro | Rembrandt74 / Paolo (#30) |
| Ice-Compatible Features | v1.0.x (Jan 2026) | Mixed | Multiple |
| Global Shortcut Conflict Fix | v1.0.3 | Free | u/a_tsygankov |
| Cmd+Drag Reveal | v1.0.x | Free | Multiple |

### Additional Features Shipped (Not Originally Requested)

These were built proactively, not from user requests:

| Feature | Tier | Description |
|---------|------|-------------|
| Always-Hidden Zone | Pro | Third zone for icons that should never appear |
| Per-Icon Hotkeys | Pro | Global keyboard shortcut per icon |
| Advanced Triggers | Pro | Battery, Wi-Fi, Focus Mode, app launch, schedule, shell script |
| Hover Trigger | Free | Show icons on mouse hover near top edge |
| Touch ID / Password Protection | Pro | Biometric auth to reveal hidden icons |
| Settings Profiles | Pro | Save/restore named configurations |
| Bartender & Ice Import | Pro | Migrate settings from competitors |
| Custom Menu Bar Icon | Pro | User-uploaded icon image |
| Liquid Glass Overlay | Pro | macOS 26+ Liquid Glass material over menu bar |
| AppleScript Automation | Pro | Scriptable commands for power users |
| Space Analyzer | Free | Debug tool showing menu bar space utilization |

---

## Sources

| Source | How to Check | Last Checked |
|--------|-------------|--------------|
| GitHub Issues | `gh issue list` | Mar 4, 2026 |
| Reddit r/macapps | Search "SaneBar" | Jan 2026 |
| Discord | Manual check | Jan 2026 |
| Support email | `check-inbox.sh check` | Mar 4, 2026 |

---

## Decision Log

| Date | Feature | Decision | Rationale |
|------|---------|----------|-----------|
| Jan 2026 | Menu Bar Spacing | Shipped | High demand, notch recovery |
| Jan 2026 | Visual Zones | Shipped | Low effort, high ROI |
| Jan 2026 | Ice-Compatible Features | Shipped | Migration path for Ice users |
| Jan 2026 | Secondary Menu Bar | Initially deprioritized | Later shipped in v2.x |
| Jan 2026 | Third-Party Overlay Detection | Rejected | Too niche, NSStatusItem positioning is macOS-controlled |
| Jan 2026 | Intel Support | Rejected | Dead platform, no test hardware |
| Mar 2026 | Bulk Icon Moves | Rejected | Fragile AXUIElement batch ops, no external demand |
| Mar 2026 | Auto-Hide App Menus (#103) | Investigating | Ice has it, need API research |

---

## Testimonials

| User | Quote | Source |
|------|-------|--------|
| ujc-cjw | "Finally, a replacement app has arrived — I'm so glad! It's been working perfectly so far." | Reddit r/macapps |
| Bernard Le Du, VVMac | "I use SaneBar daily." | Email (Feb 11, 2026) |
| u/a_tsygankov | "I really like that I can adjust SaneBar's behavior with AppleScript" | Reddit r/macapps |
