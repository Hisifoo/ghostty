# libghostty & iOS Porting Status

## Executive Summary

Ghostty already contains significant iOS infrastructure, and the maintainer (mitchellh) has confirmed that while the team **will not build an iPad/iOS app directly**, they plan to make **libghostty available for iOS/iPadOS** so community members can build apps.

---

## 1. libghostty Status

### What is libghostty?

libghostty is a C-compatible library for embedding a fast, feature-rich terminal emulator in third-party projects. It's being developed incrementally as part of the Ghostty project.

**Key Locations:**
| File | Description |
|------|-------------|
| `include/ghostty.h` | Main C API header (1,131 lines) |
| `include/ghostty/vt.h` | VT library header (87 lines) |
| `src/main_c.zig` | C API entry point |
| `src/apprt/embedded.zig` | Embedded runtime (~72KB) |

### Current Development Status

Per the Ghostty README roadmap:
> **Item #6: Cross-platform libghostty for Embeddable Terminals - WORK IN PROGRESS**

| Component | Status |
|-----------|--------|
| libghostty (full embedding) | Work in progress |
| libghostty-vt (terminal core) | Available and usable |
| API Stability | **NOT stable** - will change without warning |
| Core Logic | Well proven (extracted from production Ghostty) |

### Architecture

**Two main components:**

1. **libghostty (Main Library)**
   - Full terminal embedding API with app/surface lifecycle management
   - Provides `ghostty_app_t` and `ghostty_surface_t` opaque types
   - Built via `src/build/GhosttyLib.zig`
   - Outputs: `libghostty.so` (Linux), `libghostty.a` (static)

2. **libghostty-vt (Virtual Terminal Library)**
   - Specialized library for terminal emulation core
   - Handles: escape sequence parsing, terminal state, scrollback, reflow
   - Built via `src/build/GhosttyLibVt.zig`
   - Version: 0.1.0 (semver)
   - More portable with fewer dependencies

### Platform Support Matrix

| Platform | libghostty | libghostty-vt |
|----------|------------|---------------|
| macOS | Full (XCFramework) | Full |
| Linux (GTK) | Full | Full |
| FreeBSD | Requires GTK | Full |
| iOS | Included in XCFramework | Full |
| WebAssembly | N/A | Full (WASM module) |
| Windows | Not supported | Planned |

---

## 2. iOS Support in the Codebase

### Existing iOS Infrastructure

**iOS is already partially supported!** The codebase contains:

#### iOS App Target (Xcode Project)
- **Target**: `Ghostty-iOS` in `macos/Ghostty.xcodeproj`
- **Bundle ID**: `com.mitchellh.ghostty-ios`
- **Minimum iOS**: 17.0
- **Devices**: iPhone and iPad

#### iOS App Entry Point
**File**: `macos/Sources/App/iOS/iOSApp.swift`
```swift
@main
struct Ghostty_iOSApp: App {
    @StateObject private var ghostty_app = Ghostty.App()
    var body: some Scene {
        WindowGroup {
            iOS_GhosttyTerminal()
                .environmentObject(ghostty_app)
        }
    }
}
```

#### Platform Abstraction Layer (CrossKit)
**File**: `macos/Sources/Helpers/CrossKit.swift`

Provides platform-agnostic type aliases:
```swift
#if canImport(UIKit)
typealias OSView = UIView
typealias OSColor = UIColor
typealias OSSize = CGSize
typealias OSPasteboard = UIPasteboard
#endif
```

#### UIKit Surface View
**File**: `macos/Sources/Ghostty/SurfaceView_UIKit.swift` (132 lines)
- UIView-based terminal surface for iOS
- Uses CAMetalLayer for Metal rendering
- Handles touch input, size changes, focus events

#### iOS-Specific Code Paths
**File**: `macos/Sources/Ghostty/Ghostty.App.swift`
```swift
#if os(iOS)
// Callback implementations (currently stubs)
static func wakeup(_ userdata: UnsafeMutableRawPointer?) {}
static func action(...) -> Bool { return false }
static func readClipboard(...) {}
// ... more stubs
#endif
```

### iOS Mentions in Build System

**`src/Command.zig:350`**:
```zig
.freebsd, .ios, .macos => {
    // iOS handled alongside macOS
}
```

**`src/input/keycodes.zig:12`**:
```zig
.ios, .macos => 4, // mac keycode format
```

**`HACKING.md:53`**:
> Building the Ghostty macOS app requires that Xcode, the macOS SDK, the iOS SDK, and Metal Toolchain are all installed.

---

## 3. GitHub Discussions Summary

### Official Maintainer Position

**From [Discussion #8843](https://github.com/ghostty-org/ghostty/discussions/8843) (iPad build):**
> "We're not building an iPad app, but we plan on making libghostty available for iOS/iPadOS and others like you can build the app." — mitchellh

**From [Discussion #4087](https://github.com/ghostty-org/ghostty/discussions/4087) (Ghostty for iPad):**
> "libghostty will enable others to do this, I don't have plans currently to do this." — mitchellh

### Community Contributions

A community member (@kitknox) has developed a working iOS/iPadOS/visionOS implementation wrapping libghostty with:
- SSH support
- Scrolling and splits
- Copy/paste
- Tab management
- Proposed pipe-based backend for iOS sandbox

The maintainer indicated this pipe-based approach would be acceptable for upstream integration.

### Known Issues (Resolved)

**[Discussion #7635](https://github.com/ghostty-org/ghostty/discussions/7635) - iOS Simulator Metal shader issue:**
- Metal shaders were compiled for wrong SDK
- Fixed in PR #7636 (merged June 2025)

---

## 4. Best Approach for iOS Porting

### Recommended Strategy

1. **Use the existing iOS infrastructure** - There's already an iOS target, SwiftUI app, and UIKit surface view

2. **Implement iOS-specific callbacks** - The stub callbacks in `Ghostty.App.swift` need real implementations:
   - `readClipboard` / `writeClipboard` (use UIPasteboard)
   - `closeSurface`
   - `action` handler

3. **Handle iOS sandbox constraints**:
   - No local shell access (iOS sandbox)
   - Need pipe-based or SSH-based backend
   - Consider integrating SSH client library (e.g., libssh2)

4. **Build the XCFramework** - libghostty already builds for iOS as part of the XCFramework:
   ```bash
   zig build -Dxcframework
   ```

### Key Challenges

| Challenge | Solution |
|-----------|----------|
| No local shell | Pipe-based backend or SSH integration |
| Touch input | UIKit surface already handles basic touch |
| Keyboard | May need custom keyboard accessory view |
| Background execution | iOS background task APIs |
| App Store restrictions | Follow Apple guidelines for terminal apps |

### Files to Modify for Full iOS Support

| File | Changes Needed |
|------|----------------|
| `macos/Sources/Ghostty/Ghostty.App.swift` | Implement iOS callbacks |
| `macos/Sources/Ghostty/SurfaceView_UIKit.swift` | Enhance touch/gesture handling |
| `macos/Sources/App/iOS/iOSApp.swift` | Add UI features (settings, tabs, etc.) |
| `src/termio/` | Consider pipe-based termio backend |
| New files | SSH client integration |

---

## 5. Quick Start for iOS Development

### Building the XCFramework

```bash
# Build XCFramework (includes iOS targets)
zig build -Dxcframework

# Output location
ls macos/GhosttyKit.xcframework/
```

### Running on iOS Simulator

1. Open `macos/Ghostty.xcodeproj` in Xcode
2. Select the `Ghostty-iOS` scheme
3. Choose an iOS Simulator target
4. Build and run (Cmd+R)

Note: The iOS Simulator Metal shader issue was fixed in PR #7636 (June 2025).

---

## Sources

- [Discussion #4087: Idea: Ghostty for iPad](https://github.com/ghostty-org/ghostty/discussions/4087)
- [Discussion #8843: iPad build](https://github.com/ghostty-org/ghostty/discussions/8843)
- [Discussion #7635: iOS Simulator Metal shader issue](https://github.com/ghostty-org/ghostty/discussions/7635)
- [HACKING.md](https://github.com/ghostty-org/ghostty/blob/main/HACKING.md)
- [Ghostty README](https://github.com/ghostty-org/ghostty)

---

*Document generated: December 2025*
