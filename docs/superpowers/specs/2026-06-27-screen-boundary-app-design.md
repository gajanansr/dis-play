# Screen Boundary App — Design Spec
Date: 2026-06-27

## Problem

MacBook Air M1 has physical display damage on the right side (~2 inches from the right edge), appearing as gray/black patches. Content rendered in that region is invisible. The goal is to make macOS genuinely believe the screen ends at the safe boundary so all apps, full-screen modes, Mission Control, and the cursor are naturally constrained to the usable area.

## Target Environment

- Device: MacBook Air (Apple M1)
- Display: 2560×1600 native Retina (1280×800 logical at 2x scaling)
- macOS: 26.5.1 Tahoe (minimum target: macOS 13 Ventura)
- App type: Native Swift menu bar app, no Dock icon

## Approach

Inject a custom display mode via `CoreDisplay.framework` and `IOKit` (private/semi-private APIs, dynamically loaded) that tells the GPU to render only W pixels wide, where W = 2560 minus the damaged pixel column. macOS treats the panel as genuinely narrower — full-screen apps, screenshots, menu bar, Mission Control, and three-finger swipe all respect the boundary natively.

Private APIs are loaded via `dlsym` / `dlopen` so a symbol rename in a future macOS update causes a clean, detectable failure rather than a crash.

## Components

### 1. Display Manager
- Dynamically loads `CoreDisplay.framework` and `IOKit`
- Identifies the built-in display using `kDisplayIsBuiltIn` IOKit property (stable across reboots)
- Injects a custom `IODisplayModeID` with the target width into the IOFramebuffer service
- Switches the display to that mode via `CGDisplaySetDisplayMode` or `CGSSetDisplayMode` (SkyLight fallback)
- Exposes: `applyRestriction(pixelOffset: Int)`, `removeRestriction()`, `isRestrictionActive: Bool`
- On API failure: surfaces a `DisplayManagerError.apiUnavailable` — never silently degrades

### 2. Setup Wizard (first-run + re-open from menu bar)
- Runs at **native resolution** so the damage is visible for accurate alignment
- Full-screen dark overlay with a draggable red vertical boundary line
- Live pixel readout: shows both physical pixels (from right) and logical pixels
- Pixel input field: type a value directly, line jumps to match
- "Apply" button: applies restriction immediately as a preview
- "Confirm": saves the offset and exits wizard
- "Adjust": re-enables dragging for fine-tuning
- Saved to `UserDefaults` as `boundaryOffsetPhysicalPixels: Int`

### 3. Display Monitor
- Registers `CGDisplayReconfigurationCallback` at launch
- On any display change event:
  - If built-in only: re-apply restriction (handles wake-from-sleep and System Settings reset)
  - If any external display present: call `removeRestriction()` on built-in, leave externals untouched
  - If returning to built-in only: re-apply restriction automatically
- Debounces rapid events (100ms) to avoid flicker during reconnect

### 4. Menu Bar Controller
- Icon states:
  - Active (restriction applied): small screen icon with red right-edge bar
  - Suspended (external display connected): same icon grayed out
  - Error (API unavailable): icon with yellow warning badge
- Menu items:
  - "Screen Boundary: Active / Suspended / Error" (non-clickable status)
  - "Adjust boundary..." — reopens Setup Wizard
  - "Disable" / "Enable" toggle
  - Separator
  - "Quit"

### 5. Login Item
- Registered via `SMAppService.mainApp` (requires macOS 13+)
- On launch: reads saved `boundaryOffsetPhysicalPixels`, calls `applyRestriction()` silently
- If no saved offset found (first launch): opens Setup Wizard automatically

## Key Flows

### First launch
1. No saved offset → Setup Wizard opens at native resolution
2. User drags line to damage edge, confirms pixel value
3. App saves offset, applies restriction, enters menu bar
4. User registers as login item via menu bar (or prompted automatically)

### Every subsequent login
1. App starts, reads saved offset
2. Calls `applyRestriction()` — restriction active within ~500ms of login
3. Sits in menu bar silently

### External display connected
1. `CGDisplayReconfigurationCallback` fires
2. Display Monitor detects external display present
3. Calls `removeRestriction()` on built-in — both displays run at natural resolution
4. Menu bar icon goes gray ("Suspended")

### External display disconnected
1. Callback fires again
2. Display Monitor detects built-in only
3. Calls `applyRestriction()` — restriction restored
4. Menu bar icon goes red-active

### macOS resets display mode (sleep/wake or System Settings)
1. `kCGDisplaySetModeFlag` event fires in callback
2. Display Monitor re-applies restriction within ~1 second
3. No user action required

## Error Handling

| Scenario | Behavior |
|---|---|
| Private API symbol not found | Menu bar shows yellow warning badge, "Restriction unavailable — API changed. Update required." No silent fallback. |
| `applyRestriction()` fails at runtime | Same warning badge, error logged to Console.app under subsystem `com.screenbound.app` |
| Wrong boundary set (screen too narrow) | Global hotkey `Cmd+Ctrl+Shift+R` resets to native resolution immediately |
| First launch with no offset saved | Setup Wizard opens automatically |
| Wake from sleep resets display mode | Display Monitor auto-reapplies within 1 second |

## Recovery Hotkey

`Cmd+Ctrl+Shift+R` — registered as a global event tap at launch. Calls `removeRestriction()` regardless of app state. Works even if the menu bar is in the restricted zone.

## What Is Not Handled

- Multiple built-in displays (not possible on MacBook)
- Sidecar (iPad as display) — treated as external, no restriction applied
- Non-M1 Apple Silicon or Intel — untested, may work but not a target

## File Structure (proposed)

```
ScreenBound/
├── App/
│   ├── ScreenBoundApp.swift          # App entry point, SMAppService registration
│   └── AppDelegate.swift             # Menu bar setup
├── DisplayManager/
│   ├── DisplayManager.swift          # Public interface
│   ├── CoreDisplayBridge.swift       # Dynamic symbol loading for CoreDisplay
│   └── IOKitBridge.swift             # Custom mode injection via IOKit
├── DisplayMonitor/
│   └── DisplayMonitor.swift          # CGDisplayReconfigurationCallback handler
├── SetupWizard/
│   ├── SetupWizardView.swift         # SwiftUI full-screen overlay
│   └── BoundaryDragHandle.swift      # Draggable line component
├── MenuBar/
│   └── MenuBarController.swift       # NSStatusItem + menu
└── Shared/
    └── UserDefaultsKeys.swift        # Key constants
```

## Dependencies

None. Pure Swift + AppKit/SwiftUI + system frameworks only. No third-party packages.

## Open Questions at Implementation Time

- Exact CoreDisplay symbol names for custom mode injection on macOS 26 — verify against current framework exports via `nm /System/Library/PrivateFrameworks/CoreDisplay.framework/CoreDisplay`
- Whether `CGSSetDisplayMode` (SkyLight) or a CoreDisplay function is more stable on Apple Silicon — probe both and prefer whichever succeeds
