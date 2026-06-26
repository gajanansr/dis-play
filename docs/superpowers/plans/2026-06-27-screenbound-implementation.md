# ScreenBound Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a native Swift macOS menu bar app that restricts the MacBook Air M1's built-in display width via SkyLight private APIs so apps cannot render into the physically damaged right portion of the screen.

**Architecture:** `SkyLightBridge` dynamically loads SkyLight symbols and wraps display mode creation/configuration. `DisplayManager` finds the built-in display and orchestrates apply/remove. `DisplayMonitor` watches `CGDisplayReconfigurationCallback` and auto-reapplies on sleep/wake/external-display events. `SetupWizard` provides draggable + pixel-input boundary UI. `MenuBarController` gives status and controls. Everything is wired in `AppDelegate`.

**Tech Stack:** Swift 5.9+, AppKit, SwiftUI, CoreGraphics, IOKit, SkyLight.framework (private — loaded via `dlopen`/`dlsym`), SMAppService, CGEventTap

## Global Constraints

- Minimum deployment target: macOS 13.0 (Ventura); primary test target: macOS 26 Tahoe
- Bundle ID: `com.screenbound.app`
- `LSUIElement = YES` in Info.plist — no Dock icon
- App Sandbox: **disabled** (required for private framework access and display reconfiguration)
- Zero third-party dependencies — pure Swift + system frameworks
- All SkyLight symbols loaded via `dlopen`/`dlsym` — no `@_silgen_name` or link-time binding
- SkyLight symbols confirmed present on macOS 26: `SLDisplayModeCreate`, `SLBeginDisplayConfiguration`, `SLCompleteDisplayConfiguration`, `SLCancelDisplayConfiguration`, `SLConfigureDisplayWithDisplayMode`, `SLDisplayCopyAllDisplayModes`, `SLDisplayModeGetPixelWidth`, `SLDisplayModeGetPixelHeight`, `SLDisplayModeGetWidth`, `SLDisplayModeGetHeight`, `SLDisplayModeRelease`
- Recovery hotkey: `Cmd+Ctrl+Shift+R` always resets to native resolution

---

## File Map

```
ScreenBound/
├── ScreenBound.xcodeproj/
├── ScreenBound/
│   ├── App/
│   │   ├── ScreenBoundApp.swift          # @main, SMAppService registration
│   │   └── AppDelegate.swift             # Wires all components together
│   ├── Bridge/
│   │   └── SkyLightBridge.swift          # dlopen/dlsym + all SkyLight calls
│   ├── DisplayManager/
│   │   └── DisplayManager.swift          # applyRestriction / removeRestriction
│   ├── DisplayMonitor/
│   │   └── DisplayMonitor.swift          # CGDisplayReconfigurationCallback + debounce
│   ├── MenuBar/
│   │   └── MenuBarController.swift       # NSStatusItem + NSMenu
│   ├── SetupWizard/
│   │   ├── SetupWizardWindowController.swift
│   │   └── SetupWizardView.swift         # SwiftUI overlay with drag handle
│   ├── Shared/
│   │   └── UserDefaultsKeys.swift
│   ├── ScreenBound.entitlements
│   └── Info.plist
└── ScreenBoundTests/
    ├── SkyLightBridgeTests.swift
    ├── DisplayManagerTests.swift
    └── DisplayMonitorTests.swift
```

---

### Task 1: Xcode project scaffold + SkyLight spike

**Files:**
- Create: `ScreenBound/ScreenBound.xcodeproj` (via Xcode GUI — see steps)
- Create: `ScreenBound/ScreenBound/App/ScreenBoundApp.swift`
- Create: `ScreenBound/ScreenBound/App/AppDelegate.swift`
- Create: `ScreenBound/ScreenBound/Info.plist` (modify generated one)
- Create: `ScreenBound/ScreenBound/ScreenBound.entitlements`
- Create: `ScreenBound/ScreenBound/Bridge/SkyLightBridge.swift` (spike-only version)

**Interfaces:**
- Produces: Running `.app` bundle that logs SkyLight symbol load results; confirms `SLDisplayModeCreate` does not crash in app context

- [ ] **Step 1: Create Xcode project**

  Open Xcode → File → New → Project → macOS → App.
  - Product Name: `ScreenBound`
  - Bundle Identifier: `com.screenbound.app`
  - Language: Swift
  - Interface: SwiftUI
  - Uncheck "Include Tests" (we add them manually later)
  - Save into `/Users/gajanan/dis-play/ScreenBound/`

- [ ] **Step 2: Configure deployment target and sandbox**

  In Xcode project settings:
  - Set Deployment Target → macOS 13.0
  - Go to Signing & Capabilities → remove "App Sandbox" capability entirely
  - Add capability "Hardened Runtime" (required for notarization later; doesn't affect dev)

- [ ] **Step 3: Configure Info.plist**

  Add key to `ScreenBound/Info.plist`:
  ```xml
  <key>LSUIElement</key>
  <true/>
  ```
  This removes the Dock icon.

- [ ] **Step 4: Add entitlements file**

  Create `ScreenBound/ScreenBound.entitlements`:
  ```xml
  <?xml version="1.0" encoding="UTF-8"?>
  <!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
    "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
  <plist version="1.0">
  <dict>
      <key>com.apple.security.cs.allow-dyld-environment-variables</key>
      <false/>
      <key>com.apple.security.cs.disable-library-validation</key>
      <true/>
  </dict>
  </plist>
  ```
  In Build Settings, set "Code Signing Entitlements" to `ScreenBound/ScreenBound.entitlements`.

- [ ] **Step 5: Write spike SkyLightBridge**

  Create `ScreenBound/ScreenBound/Bridge/SkyLightBridge.swift`:
  ```swift
  import Foundation
  import CoreGraphics
  import os.log

  private let log = Logger(subsystem: "com.screenbound.app", category: "SkyLightBridge")

  final class SkyLightBridge {
      static let shared = SkyLightBridge()

      private let handle: UnsafeMutableRawPointer?

      // C function pointer types — best-guess signatures; verify with lldb if crash
      typealias SLDisplayModeCreateFn = @convention(c) (UInt32, UInt32, Double, UInt32, UInt32) -> UnsafeRawPointer?
      typealias SLBeginDisplayConfigFn = @convention(c) (UnsafeMutablePointer<UnsafeMutableRawPointer?>) -> Int32
      typealias SLConfigureDisplayWithDisplayModeFn = @convention(c) (UnsafeMutableRawPointer, CGDirectDisplayID, UnsafeRawPointer, CFDictionary?) -> Int32
      typealias SLCompleteDisplayConfigFn = @convention(c) (UnsafeMutableRawPointer, UInt32) -> Int32
      typealias SLCancelDisplayConfigFn = @convention(c) (UnsafeMutableRawPointer) -> Int32
      typealias SLDisplayModeReleaseFn = @convention(c) (UnsafeRawPointer) -> Void
      typealias SLDisplayModeGetPixelWidthFn = @convention(c) (UnsafeRawPointer) -> Int32

      private let _createMode: SLDisplayModeCreateFn?
      private let _beginConfig: SLBeginDisplayConfigFn?
      private let _configureWithMode: SLConfigureDisplayWithDisplayModeFn?
      private let _completeConfig: SLCompleteDisplayConfigFn?
      private let _cancelConfig: SLCancelDisplayConfigFn?
      private let _releaseMode: SLDisplayModeReleaseFn?
      private let _getModePixelWidth: SLDisplayModeGetPixelWidthFn?

      var isAvailable: Bool { _createMode != nil && _beginConfig != nil }

      private init() {
          handle = dlopen("/System/Library/PrivateFrameworks/SkyLight.framework/SkyLight", RTLD_NOW)
          func sym<T>(_ name: String) -> T? {
              guard let h = handle, let s = dlsym(h, name) else { return nil }
              return unsafeBitCast(s, to: T.self)
          }
          _createMode = sym("SLDisplayModeCreate")
          _beginConfig = sym("SLBeginDisplayConfiguration")
          _configureWithMode = sym("SLConfigureDisplayWithDisplayMode")
          _completeConfig = sym("SLCompleteDisplayConfiguration")
          _cancelConfig = sym("SLCancelDisplayConfiguration")
          _releaseMode = sym("SLDisplayModeRelease")
          _getModePixelWidth = sym("SLDisplayModeGetPixelWidth")

          log.info("SkyLight loaded: \(self.handle != nil), isAvailable: \(self.isAvailable)")
      }

      // SPIKE: call SLDisplayModeCreate and log the result — verifies no crash in app context
      func spikeCreateMode(pixelWidth: UInt32, pixelHeight: UInt32) {
          guard let create = _createMode else {
              log.error("SLDisplayModeCreate not found")
              return
          }
          log.info("Calling SLDisplayModeCreate(\(pixelWidth), \(pixelHeight), 60.0, 0, 0)")
          let mode = create(pixelWidth, pixelHeight, 60.0, 0, 0)
          log.info("SLDisplayModeCreate returned: \(mode != nil ? "non-nil (success)" : "nil")")
          if let mode = mode, let getW = _getModePixelWidth {
              log.info("  mode pixelWidth: \(getW(mode))")
              _releaseMode?(mode)
          }
      }
  }
  ```

- [ ] **Step 6: Call the spike from AppDelegate**

  Replace `ScreenBound/ScreenBound/App/AppDelegate.swift`:
  ```swift
  import AppKit
  import os.log

  private let log = Logger(subsystem: "com.screenbound.app", category: "AppDelegate")

  class AppDelegate: NSObject, NSApplicationDelegate {
      func applicationDidFinishLaunching(_ notification: Notification) {
          log.info("App launched")
          SkyLightBridge.shared.spikeCreateMode(pixelWidth: 2200, pixelHeight: 1600)
      }
  }
  ```

  Update `ScreenBoundApp.swift`:
  ```swift
  import SwiftUI

  @main
  struct ScreenBoundApp: App {
      @NSApplicationDelegateAdaptor(AppDelegate.self) var delegate

      var body: some Scene {
          Settings { EmptyView() }
      }
  }
  ```

- [ ] **Step 7: Run and inspect Console.app**

  Build and run (Cmd+R). Open Console.app, filter subsystem `com.screenbound.app`.

  **Expected:** You see log lines:
  ```
  SkyLight loaded: true, isAvailable: true
  Calling SLDisplayModeCreate(2200, 1600, 60.0, 0, 0)
  SLDisplayModeCreate returned: non-nil (success)
    mode pixelWidth: 2200
  ```

  **If the app crashes** inside `spikeCreateMode`: the parameter signature is wrong. In Xcode, set a breakpoint on `SLDisplayModeCreate` and use lldb to inspect register values:
  ```
  (lldb) breakpoint set --name SLDisplayModeCreate
  (lldb) run
  (lldb) register read
  ```
  Adjust the `SLDisplayModeCreateFn` typedef until the pixel width accessor returns the value you passed in.

  **If `mode` is nil**: The function exists but rejected the parameters (e.g., display ID required as first arg). Try adding `CGMainDisplayID()` as the first parameter:
  ```swift
  typealias SLDisplayModeCreateFn = @convention(c) (CGDirectDisplayID, UInt32, UInt32, Double, UInt32, UInt32) -> UnsafeRawPointer?
  ```
  And call: `create(CGMainDisplayID(), pixelWidth, pixelHeight, 60.0, 0, 0)`

- [ ] **Step 8: Commit**

  ```bash
  cd /Users/gajanan/dis-play
  git add ScreenBound/
  git commit -m "feat: scaffold ScreenBound Xcode project with SkyLight spike"
  ```

---

### Task 2: Shared constants and error types

**Files:**
- Create: `ScreenBound/ScreenBound/Shared/UserDefaultsKeys.swift`

**Interfaces:**
- Produces: `UserDefaultsKeys.boundaryOffsetPhysicalPixels: String`, `DisplayManagerError` enum

- [ ] **Step 1: Create UserDefaultsKeys.swift**

  ```swift
  import Foundation

  enum UserDefaultsKeys {
      static let boundaryOffsetPhysicalPixels = "boundaryOffsetPhysicalPixels"
  }

  enum DisplayManagerError: Error, LocalizedError {
      case apiUnavailable
      case builtInDisplayNotFound
      case modeCreationFailed
      case configurationFailed(Int32)

      var errorDescription: String? {
          switch self {
          case .apiUnavailable:
              return "SkyLight private API unavailable — macOS update may have changed it."
          case .builtInDisplayNotFound:
              return "Built-in display not found."
          case .modeCreationFailed:
              return "Failed to create custom display mode."
          case .configurationFailed(let code):
              return "Display configuration failed with error \(code)."
          }
      }
  }
  ```

- [ ] **Step 2: Add XCTest target**

  In Xcode: File → New → Target → macOS → Unit Testing Bundle.
  - Product Name: `ScreenBoundTests`
  - Target to be tested: `ScreenBound`

- [ ] **Step 3: Write tests**

  Create `ScreenBoundTests/SharedTests.swift`:
  ```swift
  import XCTest
  @testable import ScreenBound

  final class SharedTests: XCTestCase {
      func test_userDefaultsKey_isStableString() {
          XCTAssertEqual(UserDefaultsKeys.boundaryOffsetPhysicalPixels,
                         "boundaryOffsetPhysicalPixels")
      }

      func test_displayManagerError_hasDescriptions() {
          XCTAssertNotNil(DisplayManagerError.apiUnavailable.errorDescription)
          XCTAssertNotNil(DisplayManagerError.builtInDisplayNotFound.errorDescription)
          XCTAssertNotNil(DisplayManagerError.modeCreationFailed.errorDescription)
          XCTAssertNotNil(DisplayManagerError.configurationFailed(-1).errorDescription)
      }
  }
  ```

- [ ] **Step 4: Run tests**

  Cmd+U. Expected: 2 tests pass.

- [ ] **Step 5: Commit**

  ```bash
  git add ScreenBound/ScreenBound/Shared/ ScreenBoundTests/
  git commit -m "feat: add shared error types and UserDefaultsKeys"
  ```

---

### Task 3: SkyLightBridge — full implementation

**Files:**
- Modify: `ScreenBound/ScreenBound/Bridge/SkyLightBridge.swift` (replace spike with full version)
- Create: `ScreenBoundTests/SkyLightBridgeTests.swift`

**Interfaces:**
- Produces:
  - `SkyLightBridge.shared.isAvailable: Bool`
  - `SkyLightBridge.shared.createMode(pixelWidth: UInt32, pixelHeight: UInt32, logicalWidth: UInt32, logicalHeight: UInt32) throws -> UnsafeRawPointer`
  - `SkyLightBridge.shared.applyMode(_ mode: UnsafeRawPointer, to displayID: CGDirectDisplayID) throws`
  - `SkyLightBridge.shared.releaseMode(_ mode: UnsafeRawPointer)`

**Note:** Update the `SLDisplayModeCreateFn` typedef if Task 1 Step 7 revealed a different signature.

- [ ] **Step 1: Write failing tests first**

  Create `ScreenBoundTests/SkyLightBridgeTests.swift`:
  ```swift
  import XCTest
  import CoreGraphics
  @testable import ScreenBound

  final class SkyLightBridgeTests: XCTestCase {
      func test_bridge_isAvailable() {
          // SkyLight must be loadable on macOS 13+
          XCTAssertTrue(SkyLightBridge.shared.isAvailable,
                        "SkyLight private APIs not available — check macOS version or symbol names")
      }

      func test_createMode_returnsNonNilForValidDimensions() throws {
          let bridge = SkyLightBridge.shared
          guard bridge.isAvailable else {
              throw XCTSkip("SkyLight unavailable")
          }
          let mode = try bridge.createMode(pixelWidth: 2200, pixelHeight: 1600,
                                           logicalWidth: 1100, logicalHeight: 800)
          // If we get here without throwing, mode is valid
          bridge.releaseMode(mode)
      }

      func test_createMode_throwsForZeroDimensions() {
          let bridge = SkyLightBridge.shared
          guard bridge.isAvailable else { return }
          XCTAssertThrowsError(
              try bridge.createMode(pixelWidth: 0, pixelHeight: 0,
                                    logicalWidth: 0, logicalHeight: 0)
          )
      }
  }
  ```

- [ ] **Step 2: Run tests to confirm failure**

  Cmd+U. Expected: `test_createMode_returnsNonNilForValidDimensions` and `test_createMode_throwsForZeroDimensions` fail with "not implemented".

- [ ] **Step 3: Replace SkyLightBridge.swift with full implementation**

  ```swift
  import Foundation
  import CoreGraphics
  import os.log

  private let log = Logger(subsystem: "com.screenbound.app", category: "SkyLightBridge")

  final class SkyLightBridge {
      static let shared = SkyLightBridge()

      private let handle: UnsafeMutableRawPointer?

      // Adjust SLDisplayModeCreateFn if Task 1 spike revealed a different signature.
      // Current best-guess: (pixelWidth, pixelHeight, refreshRate, logicalWidth, logicalHeight)
      // If first param is CGDirectDisplayID, prepend that.
      private typealias SLDisplayModeCreateFn =
          @convention(c) (UInt32, UInt32, Double, UInt32, UInt32) -> UnsafeRawPointer?
      private typealias SLBeginDisplayConfigFn =
          @convention(c) (UnsafeMutablePointer<UnsafeMutableRawPointer?>) -> Int32
      private typealias SLConfigureDisplayWithDisplayModeFn =
          @convention(c) (UnsafeMutableRawPointer, CGDirectDisplayID, UnsafeRawPointer, CFDictionary?) -> Int32
      private typealias SLCompleteDisplayConfigFn =
          @convention(c) (UnsafeMutableRawPointer, UInt32) -> Int32
      private typealias SLCancelDisplayConfigFn =
          @convention(c) (UnsafeMutableRawPointer) -> Int32
      private typealias SLDisplayModeReleaseFn =
          @convention(c) (UnsafeRawPointer) -> Void
      private typealias SLDisplayModeGetPixelWidthFn =
          @convention(c) (UnsafeRawPointer) -> Int32

      private let _createMode: SLDisplayModeCreateFn?
      private let _beginConfig: SLBeginDisplayConfigFn?
      private let _configureWithMode: SLConfigureDisplayWithDisplayModeFn?
      private let _completeConfig: SLCompleteDisplayConfigFn?
      private let _cancelConfig: SLCancelDisplayConfigFn?
      private let _releaseMode: SLDisplayModeReleaseFn?
      private let _getModePixelWidth: SLDisplayModeGetPixelWidthFn?

      var isAvailable: Bool {
          _createMode != nil &&
          _beginConfig != nil &&
          _configureWithMode != nil &&
          _completeConfig != nil &&
          _cancelConfig != nil
      }

      private init() {
          handle = dlopen(
              "/System/Library/PrivateFrameworks/SkyLight.framework/SkyLight",
              RTLD_NOW
          )
          if handle == nil {
              log.error("dlopen SkyLight failed: \(String(cString: dlerror()))")
          }
          func sym<T>(_ name: String) -> T? {
              guard let h = handle, let s = dlsym(h, name) else {
                  log.warning("Symbol not found: \(name)")
                  return nil
              }
              return unsafeBitCast(s, to: T.self)
          }
          _createMode = sym("SLDisplayModeCreate")
          _beginConfig = sym("SLBeginDisplayConfiguration")
          _configureWithMode = sym("SLConfigureDisplayWithDisplayMode")
          _completeConfig = sym("SLCompleteDisplayConfiguration")
          _cancelConfig = sym("SLCancelDisplayConfiguration")
          _releaseMode = sym("SLDisplayModeRelease")
          _getModePixelWidth = sym("SLDisplayModeGetPixelWidth")
          log.info("SkyLightBridge init — isAvailable: \(self.isAvailable)")
      }

      /// Creates a custom display mode. Caller is responsible for calling releaseMode(_:).
      func createMode(
          pixelWidth: UInt32,
          pixelHeight: UInt32,
          logicalWidth: UInt32,
          logicalHeight: UInt32
      ) throws -> UnsafeRawPointer {
          guard let create = _createMode else { throw DisplayManagerError.apiUnavailable }
          guard pixelWidth > 0, pixelHeight > 0 else { throw DisplayManagerError.modeCreationFailed }
          // Parameters: pixelWidth, pixelHeight, refreshRate, logicalWidth, logicalHeight
          // Update order here if Task 1 spike revealed a different signature.
          guard let mode = create(pixelWidth, pixelHeight, 60.0, logicalWidth, logicalHeight) else {
              throw DisplayManagerError.modeCreationFailed
          }
          log.info("Created mode: \(pixelWidth)x\(pixelHeight) → pointer \(mode)")
          return mode
      }

      /// Applies a previously-created mode to the given display.
      func applyMode(_ mode: UnsafeRawPointer, to displayID: CGDirectDisplayID) throws {
          guard let begin = _beginConfig,
                let configure = _configureWithMode,
                let complete = _completeConfig,
                let cancel = _cancelConfig
          else { throw DisplayManagerError.apiUnavailable }

          var config: UnsafeMutableRawPointer? = nil
          let beginErr = begin(&config)
          guard beginErr == 0, let config = config else {
              throw DisplayManagerError.configurationFailed(beginErr)
          }

          let configErr = configure(config, displayID, mode, nil)
          guard configErr == 0 else {
              _ = cancel(config)
              throw DisplayManagerError.configurationFailed(configErr)
          }

          let completeErr = complete(config, 0)  // 0 = kCGConfigurePermanently equivalent
          guard completeErr == 0 else {
              throw DisplayManagerError.configurationFailed(completeErr)
          }
          log.info("Applied mode to display \(displayID)")
      }

      func releaseMode(_ mode: UnsafeRawPointer) {
          _releaseMode?(mode)
      }
  }
  ```

- [ ] **Step 4: Run tests**

  Cmd+U. Expected: all 3 `SkyLightBridgeTests` pass.

  If `test_createMode_returnsNonNilForValidDimensions` fails (mode is nil): go back to Task 1 Step 7 and determine the correct signature, update `SLDisplayModeCreateFn` accordingly.

- [ ] **Step 5: Commit**

  ```bash
  git add ScreenBound/ScreenBound/Bridge/ ScreenBoundTests/SkyLightBridgeTests.swift
  git commit -m "feat: implement SkyLightBridge with dynamic symbol loading"
  ```

---

### Task 4: DisplayManager

**Files:**
- Create: `ScreenBound/ScreenBound/DisplayManager/DisplayManager.swift`
- Create: `ScreenBoundTests/DisplayManagerTests.swift`

**Interfaces:**
- Consumes: `SkyLightBridge.shared.createMode(...)`, `SkyLightBridge.shared.applyMode(...)`, `DisplayManagerError`
- Produces:
  - `DisplayManager.shared.applyRestriction(pixelOffset: Int) throws`
  - `DisplayManager.shared.removeRestriction() throws`
  - `DisplayManager.shared.isRestrictionActive: Bool`
  - `DisplayManager.shared.builtInDisplayID() -> CGDirectDisplayID?`

- [ ] **Step 1: Write failing tests**

  Create `ScreenBoundTests/DisplayManagerTests.swift`:
  ```swift
  import XCTest
  import CoreGraphics
  @testable import ScreenBound

  final class DisplayManagerTests: XCTestCase {
      func test_builtInDisplayID_returnsValue() {
          let id = DisplayManager.shared.builtInDisplayID()
          XCTAssertNotNil(id, "Should find built-in display on MacBook")
      }

      func test_isRestrictionActive_isFalseInitially() {
          XCTAssertFalse(DisplayManager.shared.isRestrictionActive)
      }

      func test_safeWidth_calculatesFromOffset() {
          // Native width 2560, offset 400 → safe width 2160
          let safeWidth = DisplayManager.shared.safePixelWidth(nativeWidth: 2560, pixelOffset: 400)
          XCTAssertEqual(safeWidth, 2160)
      }

      func test_safeWidth_clampsToMinimum() {
          // Offset larger than screen → clamped to 400 (minimum safe)
          let safeWidth = DisplayManager.shared.safePixelWidth(nativeWidth: 2560, pixelOffset: 3000)
          XCTAssertEqual(safeWidth, 400)
      }
  }
  ```

- [ ] **Step 2: Run tests to confirm failure**

  Cmd+U. Expected: compile error (DisplayManager not defined).

- [ ] **Step 3: Implement DisplayManager.swift**

  ```swift
  import Foundation
  import CoreGraphics
  import IOKit
  import os.log

  private let log = Logger(subsystem: "com.screenbound.app", category: "DisplayManager")

  final class DisplayManager {
      static let shared = DisplayManager()

      private(set) var isRestrictionActive = false
      private var activeMode: UnsafeRawPointer?
      private var nativeMode: CGDisplayMode?

      private init() {}

      // MARK: - Public API

      func applyRestriction(pixelOffset: Int) throws {
          guard SkyLightBridge.shared.isAvailable else {
              throw DisplayManagerError.apiUnavailable
          }
          guard let displayID = builtInDisplayID() else {
              throw DisplayManagerError.builtInDisplayNotFound
          }

          // Save native mode for restore
          nativeMode = CGDisplayCopyDisplayMode(displayID)

          let nativeW = Int(CGDisplayPixelsWide(displayID))
          let nativeH = Int(CGDisplayPixelsHigh(displayID))
          let safeW = safePixelWidth(nativeWidth: nativeW, pixelOffset: pixelOffset)
          // Maintain aspect-proportional logical size at 2x Retina scaling
          let logicalW = UInt32(safeW / 2)
          let logicalH = UInt32(nativeH / 2)

          let mode = try SkyLightBridge.shared.createMode(
              pixelWidth: UInt32(safeW),
              pixelHeight: UInt32(nativeH),
              logicalWidth: logicalW,
              logicalHeight: logicalH
          )

          // Release any previously active custom mode
          if let prev = activeMode {
              SkyLightBridge.shared.releaseMode(prev)
          }
          activeMode = mode

          try SkyLightBridge.shared.applyMode(mode, to: displayID)
          isRestrictionActive = true
          log.info("Restriction applied: offset=\(pixelOffset) safeWidth=\(safeW)")
      }

      func removeRestriction() throws {
          guard let displayID = builtInDisplayID() else {
              throw DisplayManagerError.builtInDisplayNotFound
          }
          guard let native = nativeMode else {
              // Nothing to restore — already at native
              isRestrictionActive = false
              return
          }
          let err = CGDisplaySetDisplayMode(displayID, native, nil)
          guard err == .success else {
              throw DisplayManagerError.configurationFailed(Int32(err.rawValue))
          }
          if let mode = activeMode {
              SkyLightBridge.shared.releaseMode(mode)
              activeMode = nil
          }
          isRestrictionActive = false
          log.info("Restriction removed, display restored to native mode")
      }

      // MARK: - Helpers (internal for testing)

      func builtInDisplayID() -> CGDirectDisplayID? {
          var ids = [CGDirectDisplayID](repeating: 0, count: 8)
          var count: UInt32 = 0
          CGGetActiveDisplayList(8, &ids, &count)
          return ids.prefix(Int(count)).first { CGDisplayIsBuiltin($0) != 0 }
      }

      func safePixelWidth(nativeWidth: Int, pixelOffset: Int) -> Int {
          let minimum = 400  // never restrict below 400 physical pixels
          return max(nativeWidth - pixelOffset, minimum)
      }
  }
  ```

- [ ] **Step 4: Run tests**

  Cmd+U. Expected: 4 tests pass.

- [ ] **Step 5: Manual smoke test**

  Add a temporary call in `AppDelegate.applicationDidFinishLaunching`:
  ```swift
  // Temporary: test apply + remove (restores automatically after 3 seconds)
  DispatchQueue.main.asyncAfter(deadline: .now() + 1) {
      try? DisplayManager.shared.applyRestriction(pixelOffset: 400)
      DispatchQueue.main.asyncAfter(deadline: .now() + 3) {
          try? DisplayManager.shared.removeRestriction()
      }
  }
  ```
  Run the app. After 1 second the display should narrow. After 3 more seconds it returns to normal.
  Remove the temporary code afterward.

- [ ] **Step 6: Commit**

  ```bash
  git add ScreenBound/ScreenBound/DisplayManager/ ScreenBoundTests/DisplayManagerTests.swift
  git commit -m "feat: implement DisplayManager with apply/remove restriction"
  ```

---

### Task 5: DisplayMonitor

**Files:**
- Create: `ScreenBound/ScreenBound/DisplayMonitor/DisplayMonitor.swift`
- Create: `ScreenBoundTests/DisplayMonitorTests.swift`

**Interfaces:**
- Consumes: `DisplayManager.shared.applyRestriction(pixelOffset:)`, `DisplayManager.shared.removeRestriction()`
- Produces:
  - `DisplayMonitor` class with `start()` and `stop()`
  - `DisplayMonitorDelegate` protocol with `displayMonitor(_:stateChanged:)`
  - `DisplayMonitor.State` enum: `.active`, `.suspended`, `.error(Error)`

- [ ] **Step 1: Write failing tests**

  Create `ScreenBoundTests/DisplayMonitorTests.swift`:
  ```swift
  import XCTest
  @testable import ScreenBound

  final class DisplayMonitorTests: XCTestCase {
      func test_hasExternalDisplay_falseWhenBuiltInOnly() {
          let monitor = DisplayMonitor()
          // On a MacBook with no external display attached this should be false
          // (Skip this test on CI or desktop Macs)
          if CGDisplayIsBuiltin(CGMainDisplayID()) != 0 {
              XCTAssertFalse(monitor.hasExternalDisplay())
          }
      }

      func test_debounce_delaysCallback() {
          let monitor = DisplayMonitor()
          var callCount = 0
          monitor.onDebounced = { callCount += 1 }
          monitor.scheduleDebounced()
          monitor.scheduleDebounced()
          monitor.scheduleDebounced()
          // Immediately: no call yet
          XCTAssertEqual(callCount, 0)
          // After 200ms: exactly one call
          let exp = expectation(description: "debounce fires once")
          DispatchQueue.main.asyncAfter(deadline: .now() + 0.2) {
              XCTAssertEqual(callCount, 1)
              exp.fulfill()
          }
          wait(for: [exp], timeout: 1.0)
      }
  }
  ```

- [ ] **Step 2: Run tests to confirm failure**

  Cmd+U. Expected: compile error.

- [ ] **Step 3: Implement DisplayMonitor.swift**

  ```swift
  import Foundation
  import CoreGraphics
  import os.log

  private let log = Logger(subsystem: "com.screenbound.app", category: "DisplayMonitor")

  enum DisplayMonitorState {
      case active
      case suspended
      case error(Error)
  }

  protocol DisplayMonitorDelegate: AnyObject {
      func displayMonitor(_ monitor: DisplayMonitor, stateChanged state: DisplayMonitorState)
  }

  final class DisplayMonitor {
      weak var delegate: DisplayMonitorDelegate?
      var onDebounced: (() -> Void)?  // for testing

      private var debounceTimer: DispatchSourceTimer?
      private let debounceDelay: DispatchTimeInterval = .milliseconds(100)

      func start() {
          CGDisplayRegisterReconfigurationCallback(reconfigurationCallback, Unmanaged.passRetained(self).toOpaque())
          log.info("DisplayMonitor started")
      }

      func stop() {
          CGDisplayRemoveReconfigurationCallback(reconfigurationCallback, Unmanaged.passRetained(self).toOpaque())
          debounceTimer?.cancel()
          log.info("DisplayMonitor stopped")
      }

      func hasExternalDisplay() -> Bool {
          var ids = [CGDirectDisplayID](repeating: 0, count: 8)
          var count: UInt32 = 0
          CGGetActiveDisplayList(8, &ids, &count)
          return ids.prefix(Int(count)).contains { CGDisplayIsBuiltin($0) == 0 }
      }

      func scheduleDebounced() {
          debounceTimer?.cancel()
          let timer = DispatchSource.makeTimerSource(queue: .main)
          timer.schedule(deadline: .now() + debounceDelay)
          timer.setEventHandler { [weak self] in
              self?.handleDisplayChange()
              self?.onDebounced?()
          }
          timer.resume()
          debounceTimer = timer
      }

      private func handleDisplayChange() {
          let pixelOffset = UserDefaults.standard.integer(forKey: UserDefaultsKeys.boundaryOffsetPhysicalPixels)
          guard pixelOffset > 0 else { return }

          if hasExternalDisplay() {
              do {
                  try DisplayManager.shared.removeRestriction()
                  delegate?.displayMonitor(self, stateChanged: .suspended)
                  log.info("External display detected — restriction suspended")
              } catch {
                  delegate?.displayMonitor(self, stateChanged: .error(error))
              }
          } else {
              do {
                  try DisplayManager.shared.applyRestriction(pixelOffset: pixelOffset)
                  delegate?.displayMonitor(self, stateChanged: .active)
                  log.info("Built-in only — restriction re-applied")
              } catch {
                  delegate?.displayMonitor(self, stateChanged: .error(error))
                  log.error("Failed to re-apply restriction: \(error)")
              }
          }
      }
  }

  private func reconfigurationCallback(
      display: CGDirectDisplayID,
      flags: CGDisplayChangeSummaryFlags,
      userInfo: UnsafeMutableRawPointer?
  ) {
      guard let ptr = userInfo else { return }
      let monitor = Unmanaged<DisplayMonitor>.fromOpaque(ptr).takeUnretainedValue()
      // Only act on meaningful change events
      let relevantFlags: CGDisplayChangeSummaryFlags = [
          .setModeFlag, .addFlag, .removeFlag, .enabledFlag, .disabledFlag
      ]
      guard !flags.intersection(relevantFlags).isEmpty else { return }
      monitor.scheduleDebounced()
  }
  ```

- [ ] **Step 4: Run tests**

  Cmd+U. Expected: 2 tests pass (or 1 passes and 1 is skipped on a setup without external display).

- [ ] **Step 5: Commit**

  ```bash
  git add ScreenBound/ScreenBound/DisplayMonitor/ ScreenBoundTests/DisplayMonitorTests.swift
  git commit -m "feat: implement DisplayMonitor with debounced display reconfiguration callback"
  ```

---

### Task 6: Recovery hotkey

**Files:**
- Create: `ScreenBound/ScreenBound/DisplayManager/RecoveryHotkey.swift`

**Interfaces:**
- Produces: `RecoveryHotkey.register()` — registers Cmd+Ctrl+Shift+R global event tap that calls `DisplayManager.shared.removeRestriction()`

- [ ] **Step 1: Implement RecoveryHotkey.swift**

  ```swift
  import CoreGraphics
  import AppKit
  import os.log

  private let log = Logger(subsystem: "com.screenbound.app", category: "RecoveryHotkey")

  final class RecoveryHotkey {
      private var eventTap: CFMachPort?

      func register() {
          guard AXIsProcessTrusted() else {
              log.warning("Accessibility not granted — recovery hotkey unavailable")
              return
          }
          let mask = CGEventMask(1 << CGEventType.keyDown.rawValue)
          let tap = CGEvent.tapCreate(
              tap: .cgSessionEventTap,
              place: .headInsertEventTap,
              options: .defaultTap,
              eventsOfInterest: mask,
              callback: hotkeyCallback,
              userInfo: Unmanaged.passRetained(self).toOpaque()
          )
          guard let tap = tap else {
              log.error("Failed to create event tap")
              return
          }
          eventTap = tap
          let source = CFMachPortCreateRunLoopSource(nil, tap, 0)
          CFRunLoopAddSource(CFRunLoopGetMain(), source, .commonModes)
          CGEvent.tapEnable(tap: tap, enable: true)
          log.info("Recovery hotkey registered (Cmd+Ctrl+Shift+R)")
      }
  }

  private func hotkeyCallback(
      proxy: CGEventTapProxy,
      type: CGEventType,
      event: CGEvent,
      refcon: UnsafeMutableRawPointer?
  ) -> Unmanaged<CGEvent>? {
      guard type == .keyDown else { return Unmanaged.passRetained(event) }
      let keyCode = event.getIntegerValueField(.keyboardEventKeycode)
      let flags = event.flags
      // keyCode 15 = 'r'; check Cmd+Ctrl+Shift
      let required: CGEventFlags = [.maskCommand, .maskControl, .maskShift]
      if keyCode == 15 && flags.intersection(required) == required {
          log.info("Recovery hotkey triggered — removing restriction")
          DispatchQueue.main.async {
              try? DisplayManager.shared.removeRestriction()
          }
          return nil  // swallow the event
      }
      return Unmanaged.passRetained(event)
  }
  ```

- [ ] **Step 2: Manual test**

  Add `RecoveryHotkey().register()` in `AppDelegate.applicationDidFinishLaunching`, apply a restriction, then press Cmd+Ctrl+Shift+R. Display should return to native.

  **Note:** The first time, macOS will ask for Accessibility permission. Approve it in System Settings → Privacy & Security → Accessibility.

- [ ] **Step 3: Commit**

  ```bash
  git add ScreenBound/ScreenBound/DisplayManager/RecoveryHotkey.swift
  git commit -m "feat: add recovery hotkey Cmd+Ctrl+Shift+R via CGEventTap"
  ```

---

### Task 7: MenuBarController

**Files:**
- Create: `ScreenBound/ScreenBound/MenuBar/MenuBarController.swift`

**Interfaces:**
- Consumes: `DisplayMonitorState`, `DisplayManager.shared.isRestrictionActive`
- Produces:
  - `MenuBarController` class
  - `MenuBarController.setState(_ state: DisplayMonitorState)`
  - `MenuBarController.onAdjustBoundary: (() -> Void)?`
  - `MenuBarController.onToggle: (() -> Void)?`

- [ ] **Step 1: Implement MenuBarController.swift**

  ```swift
  import AppKit
  import os.log

  private let log = Logger(subsystem: "com.screenbound.app", category: "MenuBarController")

  final class MenuBarController {
      var onAdjustBoundary: (() -> Void)?
      var onToggle: (() -> Void)?

      private let statusItem = NSStatusBar.system.statusItem(withLength: NSStatusItem.variableLength)
      private var currentState: DisplayMonitorState = .suspended

      init() {
          buildMenu()
          setState(.suspended)
      }

      func setState(_ state: DisplayMonitorState) {
          currentState = state
          DispatchQueue.main.async { [weak self] in
              self?.updateIcon(for: state)
              self?.buildMenu()
          }
      }

      // MARK: - Private

      private func updateIcon(for state: DisplayMonitorState) {
          guard let button = statusItem.button else { return }
          switch state {
          case .active:
              button.image = NSImage(systemSymbolName: "rectangle.inset.filled",
                                    accessibilityDescription: "Screen boundary active")
              button.image?.isTemplate = true
          case .suspended:
              button.image = NSImage(systemSymbolName: "rectangle.dashed",
                                    accessibilityDescription: "Screen boundary suspended")
              button.image?.isTemplate = true
          case .error:
              button.image = NSImage(systemSymbolName: "exclamationmark.triangle",
                                    accessibilityDescription: "Screen boundary error")
              button.image?.isTemplate = true
          }
      }

      private func buildMenu() {
          let menu = NSMenu()

          let statusLabel: String
          switch currentState {
          case .active:   statusLabel = "Screen Boundary: Active"
          case .suspended: statusLabel = "Screen Boundary: Suspended"
          case .error:    statusLabel = "Screen Boundary: Error"
          }
          let statusItem = NSMenuItem(title: statusLabel, action: nil, keyEquivalent: "")
          statusItem.isEnabled = false
          menu.addItem(statusItem)

          if case .error(let err) = currentState {
              let errItem = NSMenuItem(title: err.localizedDescription, action: nil, keyEquivalent: "")
              errItem.isEnabled = false
              menu.addItem(errItem)
          }

          menu.addItem(.separator())

          let adjustItem = NSMenuItem(title: "Adjust boundary…", action: #selector(adjustTapped), keyEquivalent: "")
          adjustItem.target = self
          menu.addItem(adjustItem)

          let toggleTitle = DisplayManager.shared.isRestrictionActive ? "Disable" : "Enable"
          let toggleItem = NSMenuItem(title: toggleTitle, action: #selector(toggleTapped), keyEquivalent: "")
          toggleItem.target = self
          menu.addItem(toggleItem)

          menu.addItem(.separator())
          menu.addItem(NSMenuItem(title: "Quit ScreenBound", action: #selector(NSApplication.terminate(_:)), keyEquivalent: "q"))

          self.statusItem.menu = menu
      }

      @objc private func adjustTapped() { onAdjustBoundary?() }
      @objc private func toggleTapped() { onToggle?() }
  }
  ```

- [ ] **Step 2: Wire to AppDelegate temporarily**

  In `AppDelegate.applicationDidFinishLaunching`:
  ```swift
  let menuBar = MenuBarController()
  // Keep a reference so it's not deallocated
  // (will be stored on AppDelegate properly in Task 9)
  self.menuBarController = menuBar
  ```
  Add `var menuBarController: MenuBarController?` property to `AppDelegate`.

  Run the app. You should see the dashed rectangle icon in the menu bar. Clicking it should show the menu with correct items.

- [ ] **Step 3: Commit**

  ```bash
  git add ScreenBound/ScreenBound/MenuBar/
  git commit -m "feat: implement MenuBarController with three icon states and menu"
  ```

---

### Task 8: Setup Wizard

**Files:**
- Create: `ScreenBound/ScreenBound/SetupWizard/SetupWizardWindowController.swift`
- Create: `ScreenBound/ScreenBound/SetupWizard/SetupWizardView.swift`

**Interfaces:**
- Produces:
  - `SetupWizardWindowController` with `show()` method
  - `SetupWizardWindowController.onConfirm: ((Int) -> Void)?` — called with final `pixelOffset`

- [ ] **Step 1: Implement SetupWizardView.swift**

  ```swift
  import SwiftUI

  struct SetupWizardView: View {
      let screenPixelWidth: Int    // physical pixel width of built-in display
      let screenLogicalWidth: Int  // logical (point) width

      @Binding var pixelOffset: Int
      var onApply: (Int) -> Void
      var onConfirm: (Int) -> Void

      @State private var isDragging = false
      @State private var inputText: String = ""

      private var boundaryX: CGFloat {
          let fraction = 1.0 - CGFloat(pixelOffset) / CGFloat(screenPixelWidth)
          return fraction * CGFloat(screenLogicalWidth)
      }

      var body: some View {
          ZStack {
              // Dark background
              Color.black.opacity(0.85)
                  .ignoresSafeArea()

              // Damaged zone indicator (right of boundary)
              GeometryReader { geo in
                  Rectangle()
                      .fill(Color.red.opacity(0.15))
                      .frame(width: geo.size.width - boundaryX)
                      .offset(x: boundaryX)
              }

              // Draggable boundary line
              GeometryReader { geo in
                  ZStack {
                      // Line
                      Rectangle()
                          .fill(Color.red)
                          .frame(width: 2)
                          .offset(x: boundaryX - 1)

                      // Drag handle knob
                      Circle()
                          .fill(Color.red)
                          .frame(width: 24, height: 24)
                          .offset(x: boundaryX - 12, y: geo.size.height / 2 - 12)
                          .gesture(
                              DragGesture(minimumDistance: 1)
                                  .onChanged { value in
                                      isDragging = true
                                      let fraction = value.location.x / geo.size.width
                                      let newOffset = Int((1.0 - fraction) * CGFloat(screenPixelWidth))
                                      pixelOffset = max(0, min(newOffset, screenPixelWidth - 400))
                                      inputText = "\(pixelOffset)"
                                  }
                                  .onEnded { _ in isDragging = false }
                          )
                          .cursor(.resizeLeftRight)
                  }
              }

              // Info panel
              VStack(spacing: 16) {
                  Text("Drag the red line to the edge of the damaged area")
                      .font(.headline)
                      .foregroundColor(.white)

                  HStack {
                      Text("Offset from right:")
                          .foregroundColor(.white)
                      TextField("pixels", text: $inputText)
                          .textFieldStyle(.roundedBorder)
                          .frame(width: 80)
                          .onSubmit {
                              if let val = Int(inputText) {
                                  pixelOffset = max(0, min(val, screenPixelWidth - 400))
                                  inputText = "\(pixelOffset)"
                              }
                          }
                      Text("physical px  (\(pixelOffset / 2) logical)")
                          .foregroundColor(.secondary)
                  }

                  HStack(spacing: 12) {
                      Button("Apply Preview") { onApply(pixelOffset) }
                          .buttonStyle(.borderedProminent)
                          .tint(.orange)

                      Button("Confirm & Save") { onConfirm(pixelOffset) }
                          .buttonStyle(.borderedProminent)
                          .tint(.green)
                  }
              }
              .padding(24)
              .background(.ultraThinMaterial)
              .cornerRadius(12)
              .offset(y: 120)
          }
          .onAppear {
              inputText = "\(pixelOffset)"
          }
      }
  }

  extension View {
      func cursor(_ cursor: NSCursor) -> some View {
          self.onHover { inside in
              if inside { cursor.push() } else { NSCursor.pop() }
          }
      }
  }
  ```

- [ ] **Step 2: Implement SetupWizardWindowController.swift**

  ```swift
  import AppKit
  import SwiftUI
  import CoreGraphics

  final class SetupWizardWindowController: NSWindowController {
      var onConfirm: ((Int) -> Void)?

      private var currentOffset: Int = 0

      static func make() -> SetupWizardWindowController {
          // Get built-in display dimensions
          let displayID = DisplayManager.shared.builtInDisplayID() ?? CGMainDisplayID()
          let pixelW = Int(CGDisplayPixelsWide(displayID))
          let logicalW = Int(NSScreen.screens.first(where: { screen in
              let id = screen.deviceDescription[NSDeviceDescriptionKey("NSScreenNumber")] as? CGDirectDisplayID
              return id == displayID
          })?.frame.width ?? 1280)

          let savedOffset = UserDefaults.standard.integer(forKey: UserDefaultsKeys.boundaryOffsetPhysicalPixels)

          let controller = SetupWizardWindowController()
          controller.currentOffset = savedOffset > 0 ? savedOffset : pixelW / 8

          let view = SetupWizardViewWrapper(
              screenPixelWidth: pixelW,
              screenLogicalWidth: logicalW,
              initialOffset: controller.currentOffset,
              onApply: { offset in
                  controller.currentOffset = offset
                  try? DisplayManager.shared.applyRestriction(pixelOffset: offset)
              },
              onConfirm: { offset in
                  controller.currentOffset = offset
                  controller.onConfirm?(offset)
                  controller.close()
              }
          )

          let window = NSWindow(
              contentRect: NSScreen.main?.frame ?? .zero,
              styleMask: [.borderless],
              backing: .buffered,
              defer: false
          )
          window.level = .screenSaver
          window.backgroundColor = .clear
          window.isOpaque = false
          window.collectionBehavior = [.fullScreenAuxiliary, .canJoinAllSpaces]
          window.contentView = NSHostingView(rootView: view)

          controller.window = window
          return controller
      }

      func show() {
          window?.makeKeyAndOrderFront(nil)
          NSApp.activate(ignoringOtherApps: true)
      }
  }

  // SwiftUI state wrapper
  private struct SetupWizardViewWrapper: View {
      let screenPixelWidth: Int
      let screenLogicalWidth: Int
      let initialOffset: Int
      var onApply: (Int) -> Void
      var onConfirm: (Int) -> Void

      @State private var pixelOffset: Int

      init(screenPixelWidth: Int, screenLogicalWidth: Int, initialOffset: Int,
           onApply: @escaping (Int) -> Void, onConfirm: @escaping (Int) -> Void) {
          self.screenPixelWidth = screenPixelWidth
          self.screenLogicalWidth = screenLogicalWidth
          self.initialOffset = initialOffset
          self.onApply = onApply
          self.onConfirm = onConfirm
          _pixelOffset = State(initialValue: initialOffset)
      }

      var body: some View {
          SetupWizardView(
              screenPixelWidth: screenPixelWidth,
              screenLogicalWidth: screenLogicalWidth,
              pixelOffset: $pixelOffset,
              onApply: onApply,
              onConfirm: onConfirm
          )
      }
  }
  ```

- [ ] **Step 3: Manual test**

  Trigger the wizard from `AppDelegate`:
  ```swift
  let wizard = SetupWizardWindowController.make()
  wizard.onConfirm = { offset in
      print("Confirmed offset:", offset)
  }
  wizard.show()
  ```
  Verify: full-screen dark overlay appears, red line is draggable, pixel input updates the line, Apply Preview applies the restriction immediately, Confirm saves and closes.

- [ ] **Step 4: Commit**

  ```bash
  git add ScreenBound/ScreenBound/SetupWizard/
  git commit -m "feat: implement Setup Wizard with draggable boundary line and pixel input"
  ```

---

### Task 9: AppDelegate wiring + login item

**Files:**
- Modify: `ScreenBound/ScreenBound/App/AppDelegate.swift` (replace spike with full implementation)
- Modify: `ScreenBound/ScreenBound/App/ScreenBoundApp.swift`

**Interfaces:**
- Consumes: all components from Tasks 3–8
- Produces: fully working app — auto-starts, applies restriction on launch, responds to display changes, opens wizard on first run

- [ ] **Step 1: Implement full AppDelegate.swift**

  ```swift
  import AppKit
  import ServiceManagement
  import os.log

  private let log = Logger(subsystem: "com.screenbound.app", category: "AppDelegate")

  class AppDelegate: NSObject, NSApplicationDelegate {
      private var menuBarController: MenuBarController!
      private var displayMonitor: DisplayMonitor!
      private var recoveryHotkey: RecoveryHotkey!
      private var wizardController: SetupWizardWindowController?

      func applicationDidFinishLaunching(_ notification: Notification) {
          guard SkyLightBridge.shared.isAvailable else {
              showAPIUnavailableAlert()
              return
          }

          setupMenuBar()
          setupRecoveryHotkey()
          setupDisplayMonitor()
          applyOnLaunch()
          registerLoginItem()
      }

      // MARK: - Setup

      private func setupMenuBar() {
          menuBarController = MenuBarController()
          menuBarController.onAdjustBoundary = { [weak self] in self?.openWizard() }
          menuBarController.onToggle = { [weak self] in self?.toggleRestriction() }
      }

      private func setupRecoveryHotkey() {
          recoveryHotkey = RecoveryHotkey()
          recoveryHotkey.register()
      }

      private func setupDisplayMonitor() {
          displayMonitor = DisplayMonitor()
          displayMonitor.delegate = self
          displayMonitor.start()
      }

      private func applyOnLaunch() {
          let offset = UserDefaults.standard.integer(
              forKey: UserDefaultsKeys.boundaryOffsetPhysicalPixels
          )
          if offset == 0 {
              // First launch — open wizard
              DispatchQueue.main.asyncAfter(deadline: .now() + 0.5) {
                  self.openWizard()
              }
          } else {
              do {
                  try DisplayManager.shared.applyRestriction(pixelOffset: offset)
                  menuBarController.setState(.active)
              } catch {
                  log.error("Failed to apply restriction on launch: \(error)")
                  menuBarController.setState(.error(error))
              }
          }
      }

      private func registerLoginItem() {
          do {
              try SMAppService.mainApp.register()
              log.info("Registered as login item")
          } catch {
              log.warning("Failed to register login item: \(error)")
          }
      }

      // MARK: - Actions

      private func openWizard() {
          let wizard = SetupWizardWindowController.make()
          wizard.onConfirm = { [weak self] offset in
              UserDefaults.standard.set(offset, forKey: UserDefaultsKeys.boundaryOffsetPhysicalPixels)
              do {
                  try DisplayManager.shared.applyRestriction(pixelOffset: offset)
                  self?.menuBarController.setState(.active)
              } catch {
                  self?.menuBarController.setState(.error(error))
              }
          }
          wizard.show()
          wizardController = wizard
      }

      private func toggleRestriction() {
          if DisplayManager.shared.isRestrictionActive {
              do {
                  try DisplayManager.shared.removeRestriction()
                  menuBarController.setState(.suspended)
              } catch {
                  menuBarController.setState(.error(error))
              }
          } else {
              let offset = UserDefaults.standard.integer(
                  forKey: UserDefaultsKeys.boundaryOffsetPhysicalPixels
              )
              guard offset > 0 else { openWizard(); return }
              do {
                  try DisplayManager.shared.applyRestriction(pixelOffset: offset)
                  menuBarController.setState(.active)
              } catch {
                  menuBarController.setState(.error(error))
              }
          }
      }

      private func showAPIUnavailableAlert() {
          let alert = NSAlert()
          alert.messageText = "ScreenBound: API Unavailable"
          alert.informativeText = "The SkyLight private API has changed. Update required. Your display is unchanged."
          alert.alertStyle = .critical
          alert.addButton(withTitle: "Quit")
          alert.runModal()
          NSApp.terminate(nil)
      }
  }

  extension AppDelegate: DisplayMonitorDelegate {
      func displayMonitor(_ monitor: DisplayMonitor, stateChanged state: DisplayMonitorState) {
          menuBarController.setState(state)
      }
  }
  ```

- [ ] **Step 2: Update ScreenBoundApp.swift**

  ```swift
  import SwiftUI

  @main
  struct ScreenBoundApp: App {
      @NSApplicationDelegateAdaptor(AppDelegate.self) var delegate

      var body: some Scene {
          Settings { EmptyView() }
      }
  }
  ```

- [ ] **Step 3: Run all tests**

  Cmd+U. Expected: all tests pass.

- [ ] **Step 4: Full end-to-end manual test**

  Run the app clean (delete UserDefaults first):
  ```bash
  defaults delete com.screenbound.app 2>/dev/null; true
  ```
  1. App launches → Setup Wizard appears at native resolution
  2. Drag line to approximately where screen damage starts
  3. Click "Apply Preview" → screen narrows, verify boundary is correct
  4. Click "Confirm & Save" → wizard closes, menu bar icon shows active (solid rectangle)
  5. Open a full-screen app (e.g., Safari) → verify it fits within the safe boundary
  6. Three-finger swipe between spaces → works normally
  7. Press Cmd+Ctrl+Shift+R → display returns to native
  8. Connect an external monitor → menu bar icon grays out, built-in returns to native
  9. Disconnect external monitor → restriction re-applies automatically
  10. Quit and relaunch → restriction re-applies on launch without wizard

- [ ] **Step 5: Commit final**

  ```bash
  git add ScreenBound/ScreenBound/App/
  git commit -m "feat: wire all components in AppDelegate, add login item registration"
  ```

- [ ] **Step 6: Tag v0.1.0**

  ```bash
  git tag v0.1.0
  git push origin main --tags
  ```

---

## Self-Review

**Spec coverage:**
- ✅ Hard boundary via SkyLight custom mode injection
- ✅ Built-in display detection via `CGDisplayIsBuiltin`
- ✅ `applyRestriction(pixelOffset:)` / `removeRestriction()`
- ✅ Setup wizard with draggable line + pixel input
- ✅ Live pixel readout (physical + logical) in wizard
- ✅ Apply Preview + Confirm + Adjust flow
- ✅ `UserDefaults` persistence (`boundaryOffsetPhysicalPixels`)
- ✅ `CGDisplayReconfigurationCallback` in DisplayMonitor
- ✅ External display detection → auto-suspend restriction
- ✅ Return to built-in → auto-restore restriction
- ✅ 100ms debounce on display changes
- ✅ Menu bar icon: three states (active / suspended / error)
- ✅ Menu: status label, Adjust boundary, Enable/Disable, Quit
- ✅ Recovery hotkey `Cmd+Ctrl+Shift+R` via `CGEventTap`
- ✅ Login item via `SMAppService.mainApp`
- ✅ First launch detection → auto-open wizard
- ✅ API failure → alert + quit (never silently degrades)
- ✅ Error logging to Console.app under `com.screenbound.app`
- ✅ No Dock icon (`LSUIElement = YES`)
- ✅ No App Sandbox
- ✅ Zero third-party dependencies

**Gap noted:** The spec mentions the wizard runs at "native resolution" so damage is visible. The current `SetupWizardWindowController` temporarily removes the restriction before opening the wizard (not yet implemented). Add this to `openWizard()` in `AppDelegate`:
```swift
private func openWizard() {
    // Temporarily restore native so the full screen is visible during setup
    try? DisplayManager.shared.removeRestriction()
    // ... rest of wizard setup
}
```
And in `wizard.onConfirm`, apply the new restriction afterward (already done).
