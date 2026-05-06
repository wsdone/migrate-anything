# UI Framework Migration Guide

Decision framework and pitfalls for migrating between UI frameworks. The agent knows widget-level mappings — this guide focuses on paradigm shifts, architectural decisions, and non-obvious traps.

## Decision Framework

```
1. Does the project already use a cross-platform UI framework (Qt, GTK, SDL)?
   → YES: Probably already portable. Check for platform-specific integrations.
   → NO: Continue to 2

2. Is the UI deeply integrated with application logic?
   → NO (clean separation): Replace UI layer, keep logic
   → YES (tightly coupled): Continue to 3

3. How much UI code is there?
   → Small (< 20% of codebase): Rewrite UI for target platform
   → Large (> 20%): Use a cross-platform framework (Qt or GTK) and rewrite
   → Core to the app (like a terminal emulator): Rewrite is unavoidable
```

## Paradigm Differences

**Don't try to map widgets one-by-one.** Each framework has fundamentally different patterns:

### Event Handling

| Framework | Pattern | Example |
|-----------|---------|---------|
| AppKit | **Target-action** + **delegates** | `button.target = self; button.action = #selector(onClick)` |
| GTK | **Signals** + **callbacks** | `g_signal_connect(button, "clicked", callback, NULL)` |
| Qt | **Signals and slots** | `connect(button, &QPushButton::clicked, this, &MyClass::onClick)` |
| Win32 | **Message loop** + **window procedure** | `case WM_COMMAND: ...` |

### State Management

| Framework | Approach |
|-----------|----------|
| AppKit | MVC pattern, controllers own state |
| SwiftUI | Declarative, `@State` / `@Observable` |
| GTK | Composition, widgets own their state |
| Qt | MVC via `QAbstractItemModel`, or direct widget state |
| Win32 | Global/window state via message handling |

### Memory Management

| Framework | Model |
|-----------|-------|
| AppKit (Swift) | ARC (automatic) |
| AppKit (ObjC) | Manual retain/release |
| GTK | Reference counting (`g_object_ref/unref`) |
| Qt | Parent-child ownership tree (deleting parent deletes children) |
| Win32 | Manual `CreateWindow` / `DestroyWindow` |

**Getting memory management wrong causes crashes.** Understand the target framework's model before writing code.

## Key Pitfalls

### Threading and UI

**All UI frameworks require UI operations on the main thread.** There are no exceptions. To call UI code from a background thread:

| Framework | How to marshal to main thread |
|-----------|-------------------------------|
| AppKit | `DispatchQueue.main.async { ... }` |
| GTK | `g_idle_add()` or `g_main_context_invoke()` |
| Qt | `QMetaObject::invokeMethod()` with `Qt::QueuedConnection` |
| Win32 | `PostMessage()` to the UI window |

### The "Native Look" Trap

Don't try to make GTK look exactly like AppKit or vice versa. Users on each platform expect native-feeling apps. Make the migrated app feel at home on the target platform, not like a replica of the source.

### Event Loop Differences

Each framework owns the main event loop. You can't nest event loops from different frameworks. Pick one framework and commit to it.

| Framework | Event Loop |
|-----------|-----------|
| AppKit | `NSApp.run()` |
| GTK 4 | `g_application_run()` |
| Qt 6 | `QApplication.exec()` |
| Win32 | `GetMessage()` / `DispatchMessage()` loop |

### Layout Systems

Layout approaches differ significantly:

- **AppKit**: Auto Layout (constraints), springs/struts (legacy)
- **GTK 4**: Layout managers (GtkBoxLayout, GtkConstraintLayout, etc.)
- **Qt**: Layout managers (QVBoxLayout, QGridLayout, etc.)
- **Win32**: Manual positioning or dialog templates
- **SwiftUI**: Declarative stacks (VStack, HStack)

**Don't translate constraints to layout managers one-by-one.** Rethink the layout for the target framework's approach.

## The Rewrite Boundary Pattern

For Hard UI migrations (e.g., AppKit-only app → Linux):

1. **Identify what the UI provides to the application** — the interface boundary
2. **Define an abstract UI interface** — what operations does the app need?
3. **Implement for target framework** — write a new UI layer from scratch
4. **Wire it in** — swap implementations based on platform

Example boundary:
```cpp
// The interface the application uses (portable)
class UIProvider {
public:
    virtual ~UIProvider() = default;
    virtual void create_window(const WindowConfig& config) = 0;
    virtual void show_notification(const std::string& text) = 0;
    virtual void set_menu(const MenuDefinition& menu) = 0;
    virtual void run_event_loop() = 0;
};

// Factory — each platform provides its own
std::unique_ptr<UIProvider> create_ui_provider();
```

## Cross-Platform UI Library Recommendations

| Library | Language | Native Look | Best For |
|---------|----------|------------|----------|
| **Qt 6** | C++ | Yes | Feature-rich desktop apps |
| **GTK 4** | C | Yes (Linux native) | GNOME-style Linux apps |
| **SDL 3** | C | No | Games, media apps, minimal UI |
| **Dear ImGui** | C++ | No | Dev tools, editors, debug UIs |
| **wxWidgets** | C++ | Yes | Native look on all platforms |
| **Tauri** | Rust + Web | Yes | Web-tech desktop apps |
| **Flutter** | Dart | Yes | Cross-platform including mobile |

### Choosing Between Qt and GTK

- **Qt**: More complete, better documentation, commercial-friendly, larger ecosystem. Use for most desktop apps.
- **GTK**: Lighter, better Linux integration, C-native. Use for apps targeting GNOME/Linux primarily.
- **Neither**: For terminal emulators, editors, or specialized apps, consider SDL + Dear ImGui or a custom solution.
