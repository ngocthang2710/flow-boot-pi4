# ActivityManager & WindowManager — Android UX Framework

## 1. Tổng quan

```
User taps icon
  │
  ▼
Launcher → startActivity(Intent)
  │ Binder
  ▼
ActivityTaskManagerService (ATMS) — trong system_server
  │ decides process, task, back stack
  ▼
ActivityManagerService (AMS)
  │ process management, lifecycle
  ▼
App process (Zygote fork)
  │
  ▼
Activity.onCreate() → setContentView()
  │
  ▼
WindowManagerService (WMS) — window placement, Z-order
  │
  ▼
SurfaceFlinger — composites to display
```

---

## 2. Activity Lifecycle

```
         [App launches]
               │
               ▼
          onCreate()      ← Initialize UI, data
               │
               ▼
          onStart()       ← Visible but not focused
               │
               ▼
          onResume()      ← Fully visible, has input focus
               │
          [User leaves]
               │
               ▼
          onPause()       ← Partially obscured (dialog, multi-window)
               │
               ▼
          onStop()        ← Not visible
               │
        ┌──────┴──────────────────────┐
        │                             │
        ▼                             ▼
  onRestart()               onDestroy()
  onStart()               [process may be killed]
  onResume()
```

---

## 3. Task & Back Stack

```
Tasks (conceptually like browser tabs):
  Task 1: [Launcher → Settings → WiFi settings]
           ↑ back stack (LIFO)
  Task 2: [Gmail → Compose]

Back stack rules:
  startActivity() → pushes to current task
  finish() or Back → pops from stack
  
Launch modes (AndroidManifest.xml):
  standard:         New instance every time
  singleTop:        Reuse if already on top (onNewIntent)
  singleTask:       One instance per task, clears top (onNewIntent)
  singleInstance:   One instance, own task
  
Intent flags:
  FLAG_ACTIVITY_NEW_TASK         → Start in new task
  FLAG_ACTIVITY_CLEAR_TOP        → Clear all above target
  FLAG_ACTIVITY_SINGLE_TOP       → Reuse top
  FLAG_ACTIVITY_NO_HISTORY       → Don't add to back stack
```

---

## 4. ActivityTaskManagerService (ATMS)

```java
// ATMS manages "Tasks" and "Activity Records"
// Separated from AMS in Android 10

// Key data structures:
// ActivityRecord     → one Activity instance
// Task               → stack of ActivityRecords (= back stack)
// TaskDisplayArea    → display area (main, split-screen)
// RootWindowContainer → root of window hierarchy

// startActivity() path:
// Binder call → ATMS.startActivity()
//   → resolveActivity (find component from Intent)
//   → checkStartActivityPermission (SELinux, permissions)
//   → startActivityUnchecked()
//   → Task.startActivityLocked()
//   → ActivityRecord created
//   → resumeTopActivity()
//   → AMS ensures process exists (or starts it)
//   → scheduleLaunchActivity() to app process
```

---

## 5. Process Lifecycle — AMS

```
Process states (AMS):
  PROCESS_STATE_FOREGROUND_APP     = 4  ← user interaction
  PROCESS_STATE_FOREGROUND_SERVICE = 5  ← visible notification
  PROCESS_STATE_TOP_SLEEPING       = 6  ← top but screen off
  PROCESS_STATE_BOUND_FOREGROUND   = 7  ← bound to fg
  PROCESS_STATE_IMPORTANT_FOREGROUND= 8 ← important to fg
  PROCESS_STATE_IMPORTANT_BACKGROUND= 9
  PROCESS_STATE_TRANSIENT_BACKGROUND=10
  PROCESS_STATE_BACKUP             = 11
  PROCESS_STATE_SERVICE            = 12 ← running service
  PROCESS_STATE_RECEIVER           = 13 ← handling broadcast
  PROCESS_STATE_CACHED_RECENT      = 14
  PROCESS_STATE_CACHED_EMPTY       = 15
  PROCESS_STATE_NONEXISTENT        = 16 ← dead

Process state → OOM ADJ score (see 30_lmkd_oom.md)
```

---

## 6. WindowManagerService (WMS)

```
WMS manages:
  Window tokens & surfaces
  Window Z-order (layering)
  Window focus (which receives input)
  Animations (transitions between activities)
  Insets (status bar, nav bar, keyboard)

Window types (z-order):
  TYPE_BASE_APPLICATION    = 1    ← regular app windows
  TYPE_APPLICATION_PANEL   = 1000 ← sub-windows
  TYPE_INPUT_METHOD        = 2011 ← keyboard
  TYPE_STATUS_BAR          = 2000 ← top status bar
  TYPE_NAVIGATION_BAR      = 2019 ← bottom nav
  TYPE_SYSTEM_ALERT        = 2003 ← overlays
  TYPE_TOAST               = 2005 ← toast messages

WMS ↔ SurfaceFlinger:
  WMS creates SurfaceControl per window
  SF composites all SurfaceControls
```

---

## 7. Window Insets (System UI)

```java
// Handle system bar overlaps (Android 11+)
ViewCompat.setOnApplyWindowInsetsListener(view, (v, insets) -> {
    Insets systemBars = insets.getInsets(
        WindowInsetsCompat.Type.systemBars());
    v.setPadding(
        systemBars.left,
        systemBars.top,
        systemBars.right,
        systemBars.bottom
    );
    return insets;
});

// Go edge-to-edge (Android 15+)
WindowCompat.setDecorFitsSystemWindows(window, false);
```

---

## 8. App Transitions & Animations

```
Activity transitions:
  startActivity() → WMS triggers transition animation
  
  Types:
    overridePendingTransition(R.anim.fade_in, R.anim.fade_out)
    
  Shared element transitions (Material):
    ActivityOptionsCompat.makeSceneTransitionAnimation(...)
    
  Predictive back (Android 14+):
    OnBackPressedCallback → preview animation before final back
```

```bash
# Debug transitions
adb shell setprop persist.wm.debug.show_surface_alloc 1

# Slow down animations (scale = 1.0 normal, 10.0 very slow)
adb shell settings put global window_animation_scale 5.0
adb shell settings put global transition_animation_scale 5.0
adb shell settings put global animator_duration_scale 5.0

# Disable all animations
adb shell settings put global window_animation_scale 0
adb shell settings put global transition_animation_scale 0
adb shell settings put global animator_duration_scale 0
```

---

## 9. Multi-Window & Display

```
Multi-window modes:
  Split-screen:  Two apps side by side (Android 7+)
  Freeform:      Desktop-like floating windows (tablet, dev mode)
  PiP:           Picture-in-Picture (video calls, maps)
  
Pi4 = single display, but:
  Can attach second display via HDMI (second DSI display)
  Presentation class for secondary display output
```

```bash
# Enable freeform (developer option)
adb shell settings put global enable_freeform_support 1
adb shell am restart

# Force app into split-screen
adb shell am start -n com.myapp/.MainActivity \
    --activity-launch-mode split_screen

# Check multi-window support
adb shell dumpsys activity | grep -A5 "Multi-window"
```

---

## 10. Debug AMS/WMS

```bash
# Activity stack
adb shell dumpsys activity activities
# Display #0 (activities from top to bottom):
#   Task id #1234
#     * Hist #0: ActivityRecord{...} com.myapp/.MainActivity

# Running processes
adb shell dumpsys activity processes | head -50

# Window list (WMS)
adb shell dumpsys window windows
# Window #0: Window{... com.android.launcher3/...}
# Window #1: Window{... StatusBar}
# Window #2: Window{... NavigationBar}

# WMS focus
adb shell dumpsys window | grep mCurrentFocus
# mCurrentFocus=Window{... com.myapp/.MainActivity}

# Activity history
adb shell dumpsys activity history

# Force stop
adb shell am force-stop com.myapp

# Start activity from adb
adb shell am start -n com.myapp/.MainActivity
adb shell am start -a android.intent.action.VIEW \
    -d https://google.com
```

---

## 11. File quan trọng trong source

| File | Mô tả |
|------|-------|
| `frameworks/base/services/core/java/com/android/server/wm/ActivityTaskManagerService.java` | ATMS |
| `frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java` | AMS |
| `frameworks/base/services/core/java/com/android/server/wm/WindowManagerService.java` | WMS |
| `frameworks/base/services/core/java/com/android/server/wm/ActivityRecord.java` | Activity state |
| `frameworks/base/services/core/java/com/android/server/wm/Task.java` | Back stack |
| `frameworks/base/core/java/android/app/Activity.java` | App-side Activity |
| `frameworks/base/core/java/android/view/WindowManager.java` | Window API |
