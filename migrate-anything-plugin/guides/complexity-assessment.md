# Migration Complexity Assessment Guide

A structured framework for rating migration complexity. This is a **calibration tool** — use the scoring matrix and decision tree to produce honest, defensible estimates before starting any migration. The real-world examples provide reference points for each complexity tier.

---

## 1. Complexity Levels

Each migration is assigned a score from 1 to 10. The score determines the overall complexity tier.

### Simple (1-3)

Minor, mostly mechanical changes. The codebase already uses portable abstractions or has minimal platform coupling. Migration involves updating imports, swapping build flags, or replacing small shim layers. No architectural changes are required.

Characteristics:
- The project is written in a portable language (C, C++, Rust, Go, Java, Python) with few or no OS-specific APIs.
- Platform-specific code is isolated behind thin abstraction layers that can be reimplemented straightforwardly.
- The UI (if any) is built with a cross-platform toolkit or is absent (CLI, library, daemon).
- Build system changes are limited to updating target triples, compiler flags, or dependency URLs.

### Medium (4-7)

Moderate refactoring or library replacements are needed. Some architectural adjustments may be required, but the overall structure of the project can remain intact. You will likely need to find replacement libraries for several dependencies and rewrite portions of glue code.

Characteristics:
- The project uses a cross-platform UI framework (Qt, GTK, Electron) but with significant platform-specific integrations (native menus, system tray, file dialogs, platform-specific rendering paths).
- Several platform-specific APIs are used, but each has a reasonable equivalent on the target platform.
- The language is portable, but idioms or compiler-specific features need adjustment.
- Build system requires significant rework (e.g., Xcode project to CMake, MSBuild to Meson).
- Some subsystems need full reimplementation (networking layer, IPC mechanism, plugin system).

### Hard (8-10)

Major rewrites, language changes, or architecture redesigns are required. The project is deeply coupled to its source platform, language, or framework, and migrating it means fundamentally changing how large portions of the codebase work.

Characteristics:
- The project uses a native UI framework with no cross-platform equivalent (AppKit, UIKit, WPF, WinUI).
- The project depends on a platform-specific graphics API (Metal, DirectX) with no direct equivalent on the target.
- The source language is not available on the target platform (Swift on Linux, C# without .NET Core, Objective-C outside Apple platforms).
- The architecture is built around platform-specific paradigms (Grand Central Dispatch, COM, WinRT, Objective-C runtime message passing).
- More than half the codebase is platform-specific.

---

## 2. Scoring Matrix

Score each dimension from 0 to the maximum listed. Add the weighted subtotal for each row. The final score is the sum of all weighted subtotals, normalized to a 1-10 scale.

| Factor | Weight | Criteria | Score |
|---|---|---|---|
| **Platform-specific APIs** | 2x | None | 0 |
| | | 1-3 APIs, each with a direct equivalent | 1 |
| | | 4-10 APIs, equivalents exist but need adaptation | 2 |
| | | 10+ APIs, some without equivalents | 3 |
| | | Pervasive use of platform-exclusive APIs with no equivalents | 4 |
| **UI framework dependency** | 2x | No UI / terminal UI | 0 |
| | | Cross-platform UI (Qt, GTK, Electron, Flutter) | 1 |
| | | Cross-platform UI with deep native integrations | 2 |
| | | Native UI framework with partial cross-platform alternative | 3 |
| | | Native UI framework with no cross-platform alternative (AppKit, UIKit, WPF) | 4 |
| **Graphics API usage** | 2x | No graphics API usage | 0 |
| | | OpenGL / Vulkan (portable) | 1 |
| | | Cross-platform abstraction (SDL, Skia) | 1 |
| | | Metal or DirectX with portable alternative path needed | 3 |
| | | Deep Metal / DirectX integration with compute shaders, tile shading, etc. | 4 |
| **Language-specific features** | 1.5x | Fully portable language, standard idioms | 0 |
| | | Portable language with some compiler extensions | 1 |
| | | Portable language with heavy use of platform-specific idioms | 2 |
| | | Language unavailable on target (Swift to Linux, ObjC to Windows) | 3 |
| | | Language unavailable + runtime dependencies (e.g., ObjC runtime, CLR) | 4 |
| **Build system complexity** | 1x | Simple build (Makefile, Cargo, go build) | 0 |
| | | Standard build system (CMake, Meson, Gradle) | 1 |
| | | Platform-locked build system (Xcode, MSBuild) with moderate complexity | 2 |
| | | Platform-locked build system with extensive code generation, custom targets | 3 |
| **Platform-specific code (%)** | 1.5x | Less than 5% of codebase | 0 |
| | | 5-15% of codebase | 1 |
| | | 15-30% of codebase | 2 |
| | | 30-50% of codebase | 3 |
| | | More than 50% of codebase | 4 |

### Calculating the Final Score

1. For each row, multiply your Criteria Score by the Weight to get the Weighted Subtotal.
2. Sum all Weighted Subtotals. The maximum possible raw score is: (4x2) + (4x2) + (4x2) + (4x1.5) + (3x1) + (4x1.5) = 8 + 8 + 8 + 6 + 3 + 6 = 39.
3. Normalize to 1-10: `Final Score = max(1, round((Raw Score / 39) * 10))`

### Quick Reference for Final Score

| Raw Score Range | Final Score | Complexity Tier |
|---|---|---|
| 0-6 | 1-2 | Simple |
| 7-12 | 3 | Simple |
| 13-18 | 4-5 | Medium |
| 19-25 | 6-7 | Medium |
| 26-32 | 8 | Hard |
| 33-39 | 9-10 | Hard |

---

## 3. Real-World Examples

### Simple (Score 1-3)

Characterized by a small number of mechanical API replacements in a mostly-portable codebase. Platform-specific code is under 5% of the codebase.

**Examples:**
- **CLI tool with Win32 file I/O** (score 2): ~5 Win32 calls (`CreateFile`, `ReadFile`, etc.) replaced with POSIX equivalents. Single Makefile. Search-and-replace migration.
- **Python/Go/Rust portable projects** (score 1-2): Language is already portable, migration involves switching to `pathlib`, replacing a few syscalls (`epoll` → `kqueue`), or fixing cgo bindings. Trivial.

### Medium (Score 4-7)

**Qt application with platform integrations** (estimated score: 5)
A Qt-based desktop application that uses Qt for its UI but has platform-specific code for:
- System tray icon behavior (different APIs per OS)
- Native file dialogs and printing
- Platform-specific notification integration (libnotify on Linux, Growl/NotificationCenter on macOS)
- A Windows installer (NSIS) and macOS DMG packaging

Qt itself is cross-platform, but ~20% of the code is platform-specific integrations. Each integration has an equivalent on the target platform but needs research and testing. The build system is CMake with some platform-specific flags.

**Go microservice with cgo bindings to a Linux library** (estimated score: 5)
A Go service that uses cgo to call into `libsystemd` for journal logging and socket activation. Migrating to Windows means finding equivalent logging infrastructure (Windows Event Log) and service management (Windows Services API). The core business logic is portable Go.

**Electron app with native Node addons** (estimated score: 6)
An Electron application with several native Node.js addons written in C++ that wrap platform-specific libraries (macOS Keychain access, Windows Credential Store). The Electron shell is portable, but each addon needs a platform-specific reimplementation.

### Hard (Score 8-10)

**Ghostty** (estimated score: 10)
Ghostty is a GPU-accelerated terminal emulator built with:
- macOS AppKit for the entire UI layer (windows, tabs, settings, menus)
- Metal for GPU-accelerated text rendering with custom shaders
- Swift and Objective-C throughout
- Deep integration with macOS accessibility APIs, input methods, and window management
- Objective-C runtime message passing as a core architectural pattern
- Xcode as the exclusive build system with custom build phases

To migrate this to Linux or Windows, essentially every layer except the terminal emulation logic (VT parsing) would need to be rewritten. The UI would need a complete replacement framework. The Metal shaders would need conversion to Vulkan or DirectX. The Swift/ObjC code would need translation to a portable language. This is a full rewrite, not a migration.

**WPF/WinUI Windows-only application** (estimated score: 9)
A line-of-business application built with:
- WPF or WinUI 3 for the entire UI
- XAML data binding, styles, and control templates deeply integrated
- DirectX-based rendering pipeline (WPF uses DirectX internally)
- Windows-specific APIs: Registry, COM components, WCF services, Windows Authentication
- MSIX packaging and ClickOnce deployment
- C# with extensive use of Windows-only BCL APIs

WPF and WinUI have no cross-platform equivalents. The XAML UI layer must be completely rebuilt using a different framework (Qt, Avalonia, web-based). COM interop code has no equivalent outside Windows. The deployment model must be replaced entirely.

**iOS UIKit application** (estimated score: 9)
An iPhone application built with:
- UIKit for all UI (view controllers, navigation controllers, table views, gestures)
- Core Data for persistence
- CocoaPods/SPM for dependency management
- Swift with extensive use of iOS-only frameworks (CoreLocation, MapKit, HealthKit, PhotosUI)
- Storyboards and XIBs for UI layout
- Xcode as the sole build tool

UIKit has no equivalent on any other platform. Every screen must be rebuilt. iOS-only frameworks may have rough equivalents (Android equivalents, web APIs) but the integration patterns are entirely different. Swift is available on Linux but the iOS frameworks are not. This is a full rewrite on a new platform.

**macOS kernel extension migrated to eBPF on Linux** (estimated score: 8)
A macOS kext that intercepts filesystem operations and network traffic. The migration target is Linux using eBPF. The concepts overlap (kernel-level interception) but the APIs, programming model, and constraints are entirely different. eBPF has a restricted execution environment (verified, no loops, limited stack). The kext logic must be fundamentally restructured.

---

## 4. Decision Tree

Work through this tree top to bottom. The first "yes" that applies sets the floor for your complexity score. You can raise it based on additional factors, but never lower it below what the tree indicates.

```
START
 |
 |-- Does it use a native UI framework with NO cross-platform alternative?
 |   (AppKit, UIKit, WPF, WinUI, Android Views)
 |   YES --> Hard (minimum 8)
 |
 |-- Does it use a platform-specific graphics API as a core dependency?
 |   (Metal, DirectX 11/12 with no OpenGL/Vulkan fallback)
 |   YES --> Hard (minimum 8)
 |
 |-- Is it written in a language that is not available on the target platform?
 |   (Swift needing to run on Windows, ObjC on Linux, VB6 anywhere)
 |   YES --> Hard (minimum 8)
 |
 |-- Does the architecture depend on a platform-specific runtime or paradigm?
 |   (COM/WinRT, ObjC runtime message passing, Grand Central Dispatch as core)
 |   YES --> Hard (minimum 8)
 |
 |-- Does it use a cross-platform UI framework WITH deep native integrations?
 |   (Qt with native menus, system tray, platform-specific rendering)
 |   YES --> Medium (minimum 5)
 |   |
 |   |-- Are there MORE than 10 platform-specific API calls?
 |       YES --> Medium to Hard (6-8)
 |
 |-- Is it written in a portable language with SOME platform-specific code?
 |   (C/C++ with #ifdef, Rust with cfg, Java with JNI)
 |   YES --> Simple to Medium (2-5)
 |   |
 |   |-- Is platform-specific code LESS THAN 10%?
 |       YES --> Simple (2-3)
 |       NO  --> Medium (4-5)
 |
 |-- Is it written in a portable language with ONLY standard library usage?
 |   YES --> Simple (1-2)
 |
 END
```

### Additional Adjustments

After the decision tree sets the floor, apply these modifiers:

| Condition | Adjustment |
|---|---|
| No automated tests exist | +1 |
| Documentation is sparse or outdated | +1 |
| Migration requires a language change (not just platform) | +2 |
| Original authors are unavailable for questions | +1 |
| Codebase has not been actively maintained for 2+ years | +1 |
| Target platform has poor library ecosystem for the domain | +1 |
| Migration must maintain backward compatibility during transition | +1 |
| Hard real-time requirements (latency-sensitive) | +1 |

Cap the final score at 10.

---

## 5. Effort Estimation Guide

Rough estimates in developer-weeks for a single experienced developer proficient in both platforms. Adjust based on team size (diminishing returns beyond 3-4 developers) and codebase familiarity.

| Project Size | Simple (1-3) | Medium (4-7) | Hard (8-10) |
|---|---|---|---|
| **Small** (<10K LoC) | 1-2 weeks | 2-6 weeks | 6-16 weeks |
| **Medium** (10K-100K LoC) | 2-4 weeks | 6-20 weeks | 16-60 weeks |
| **Large** (>100K LoC) | 4-10 weeks | 12-40 weeks | 40-200+ weeks |

### Effort Multipliers

| Factor | Multiplier |
|---|---|
| Developer is unfamiliar with the source platform | 1.3x |
| Developer is unfamiliar with the target platform | 1.5x |
| Developer is unfamiliar with both | 2.0x |
| No existing tests, must be reverse-engineered | 1.4x |
| Must maintain feature parity during phased migration | 1.5x |
| Performance-critical application (games, real-time systems) | 1.5x |
| Regulatory/compliance requirements (medical, aerospace, finance) | 2.0x |

---

## 6. Risk Assessment

### Simple Migrations -- What Can Go Wrong

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Subtle behavioral differences in "equivalent" APIs | Medium | Low | Write comparison tests on both platforms |
| Missing edge cases in platform-specific code paths | Medium | Medium | Audit all `#ifdef` and conditional compilation blocks |
| Build system quirks on the new platform | Medium | Low | Test CI on the target platform early |
| Dependency not available on target platform | Low | Medium | Verify all dependencies before starting |

Simple migrations rarely fail catastrophically. The most common problem is spending more time than expected on small incompatibilities that add up. Budget 20% more time than the estimate.

### Medium Migrations -- What Can Go Wrong

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Platform API has no true equivalent, requires workaround | High | High | Prototype the hardest integrations before committing to timeline |
| Replacement library has different semantics or bugs | High | Medium | Spike test critical library replacements early |
| Performance regression on target platform | Medium | High | Benchmark early and often; profile on target platform |
| UI/UX feels "wrong" on the new platform | High | Medium | Involve designers or platform-native users for review |
| Scope creep as hidden platform dependencies emerge | High | High | Do a thorough API audit before estimating; maintain a discovery log |
| Test suite is platform-specific and needs porting first | Medium | Medium | Port tests before porting code |

Medium migrations are where projects most commonly underestimate effort. The initial assessment looks manageable, but each "simple replacement" hides edge cases. Budget 40% more time than the estimate and plan for at least one "discovery" phase.

### Hard Migrations -- What Can Go Wrong

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Fundamental architectural mismatch between platforms | High | Critical | Build a throwaway prototype of the hardest 10% before committing |
| Complete rewrite exceeds estimate by 2-3x | High | Critical | Break into phases with go/no-go checkpoints |
| Performance requirements cannot be met on target platform | Medium | Critical | Benchmark the critical path on the target platform before full migration |
| Feature parity is infeasible; must redesign features | High | High | Define a minimum viable migration scope with explicit exclusions |
| Team burnout on a long, grinding project | High | High | Plan breaks; consider phased delivery with user-visible milestones |
| Abandoned dependencies block progress | Medium | High | Audit dependency health on the target platform; identify alternatives early |
| Moving target (source project evolves during migration) | Medium | High | Pin to a specific source version; plan a catch-up phase |
| Legal/licensing incompatibilities discovered late | Low | Critical | Audit all licenses before starting |

Hard migrations have a significant failure rate. Treat them as new development projects that happen to have a detailed specification (the source code). Expect to write 40-80% new code. Budget 60-100% more time than the estimate and treat the estimate as a best case.

### Universal Risks (All Complexity Levels)

| Risk | Mitigation |
|---|---|
| Underestimation of effort | Apply effort multipliers honestly; pad estimates |
| Loss of institutional knowledge | Document all decisions; maintain a migration journal |
| Integration testing gaps | Set up CI on the target platform on day one |
| User/ stakeholder frustration with timeline | Communicate that migration is development, not translation |
| Regression in production | Maintain the ability to run the old version alongside the new |

---

## Appendix: Quick Assessment Checklist

Run through this checklist before giving a complexity estimate. If you cannot answer a question, that itself increases risk.

- [ ] Have I identified every platform-specific API call in the codebase?
- [ ] Do I know what each platform-specific API does and whether an equivalent exists?
- [ ] Have I checked whether all dependencies are available on the target platform?
- [ ] Have I verified the target platform's build toolchain works for this language?
- [ ] Have I estimated what percentage of the codebase is platform-specific?
- [ ] Have I identified any language features that are unavailable on the target?
- [ ] Do I understand the project's architecture well enough to know what can be ported vs. what must be rewritten?
- [ ] Have I run the scoring matrix and recorded each factor's score?
- [ ] Have I applied the decision tree and any adjustment modifiers?
- [ ] Have I checked for the common risk factors and planned mitigations?

If you answered "no" to any of these, spend time investigating before providing a final estimate. An informed estimate is always better than a fast one.
