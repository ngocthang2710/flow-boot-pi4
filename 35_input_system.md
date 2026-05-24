# Android Input System — Touch, Keys, Sensors

## 1. Tổng quan

Input system xử lý mọi sự kiện vật lý: cảm ứng, phím, chuột, bút, gamepad.

```
Hardware → Kernel driver → /dev/input/eventN → InputReader → InputDispatcher → App
```

---

## 2. Kiến trúc đầy đủ

```
Physical device (touch, keyboard, mouse)
        │  IRQ → kernel driver
        ▼
/dev/input/event0, event1, ...  (Linux input subsystem)
        │  evdev interface (struct input_event)
        ▼
EventHub (InputReader's reader)
        │  poll() trên tất cả /dev/input/
        ▼
InputReader (native thread trong system_server)
        │  Raw events → MotionEvent / KeyEvent
        │  Calibration, gesture recognition
        ▼
InputDispatcher (native thread trong system_server)
        │  Find target window
        │  Dispatch qua InputChannel (Unix socket pair)
        ▼
ViewRootImpl (app main thread)
        │  deliverInputEvent()
        ▼
View.dispatchTouchEvent() / onTouchEvent()
```

---

## 3. Linux Input Subsystem

```c
/* Kernel: struct input_event */
struct input_event {
    struct timeval time;
    __u16 type;   /* EV_KEY, EV_ABS, EV_REL, EV_SYN */
    __u16 code;   /* KEY_A, ABS_X, REL_X, ... */
    __s32 value;  /* key: 0/1/2, abs: position, rel: delta */
};

/* Touch event sequence: */
// EV_ABS ABS_MT_TRACKING_ID 1   ← Finger down (id=1)
// EV_ABS ABS_MT_POSITION_X  450 ← X coordinate
// EV_ABS ABS_MT_POSITION_Y  800 ← Y coordinate
// EV_ABS ABS_MT_PRESSURE    50  ← Pressure
// EV_SYN SYN_MT_REPORT  0       ← End of this touch
// EV_SYN SYN_REPORT     0       ← End of event frame
```

---

## 4. Đọc Input Events trực tiếp

```bash
# List input devices
adb shell getevent -l
# /dev/input/event0: EV_KEY KEY_POWER
# /dev/input/event1: EV_ABS ABS_MT_POSITION_X ABS_MT_POSITION_Y ...

# Monitor events
adb shell getevent -t /dev/input/event1
# 1234567890.123456 0003 0039 00000001  ← EV_ABS ABS_MT_TRACKING_ID 1
# 1234567890.123456 0003 0035 000001C2  ← EV_ABS ABS_MT_POSITION_X 450

# Inject events (cần root)
adb shell sendevent /dev/input/event1 3 57 1     # MT_TRACKING_ID=1
adb shell sendevent /dev/input/event1 3 53 500   # MT_POS_X=500
adb shell sendevent /dev/input/event1 3 54 800   # MT_POS_Y=800
adb shell sendevent /dev/input/event1 0 0 0      # SYN_REPORT
```

---

## 5. InputReader — Raw → MotionEvent

```cpp
// frameworks/native/services/inputflinger/reader/InputReader.cpp

void InputReader::loopOnce() {
    /* 1. Đọc events từ EventHub */
    size_t count = mEventHub->getEvents(timeoutMillis, mEventBuffer, EVENT_BUFFER_SIZE);

    /* 2. Process mỗi event */
    for (size_t i = 0; i < count; i++) {
        const RawEvent* rawEvent = &mEventBuffer[i];
        device->process(rawEvent);  /* dispatch tới đúng InputMapper */
    }
}

/* Các InputMapper xử lý event theo loại thiết bị: */
// TouchInputMapper     ← cảm ứng đa điểm
// KeyboardInputMapper  ← phím
// CursorInputMapper    ← chuột/trackpad
// SwitchInputMapper    ← power button, lid
// VibratorInputMapper  ← rung

/* TouchInputMapper convert raw ABS events → MotionEvent */
```

---

## 6. Calibration — Chuyển đổi tọa độ

```
Raw touch coords (0-4095 pixel driver)
        │  Calibration matrix (affine transform)
        ▼
Screen coords (0-1920 x 0-1080)

Config: /system/usr/idc/<device>.idc
```

```
# /system/usr/idc/touchscreen.idc
device.internal = 1
touch.deviceType = touchScreen

# Calibration
touch.size.calibration = diameter
touch.size.scale = 10

# Orientation
touch.orientationAware = 1
touch.orientation.calibration = interpolated
```

---

## 7. InputDispatcher — Routing Events

```cpp
// frameworks/native/services/inputflinger/dispatcher/InputDispatcher.cpp

void InputDispatcher::dispatchMotionLocked(MotionEntry* entry) {
    /* 1. Find target window (window dưới pointer) */
    std::vector<InputTarget> inputTargets;
    findTouchedWindowTargetsLocked(currentTime, *entry, inputTargets);

    /* 2. Check ANR timeout */
    /* Nếu window không consume event trong 5s → ANR */

    /* 3. Dispatch qua InputChannel */
    dispatchEventLocked(currentTime, entry, inputTargets);
}

/* InputChannel = Unix socket pair:
   Dispatcher (write) ←→ (read) App's ViewRootImpl */
```

---

## 8. Touch Coordinate Mapping (Pi4)

```bash
# Pi4 dùng USB HID touchscreen hoặc GPIO touch
# Calibrate: /system/usr/idc/

adb shell ls /system/usr/idc/
# Multitouch.idc  ft5x06.idc  ...

# Xem device info
adb shell getevent -p /dev/input/event1
# name:     "ADS7846 Touchscreen"
# events:
#   ABS (0003): ABS_X       : value 2048, min 0, max 4095, fuzz 0, flat 0, resolution 0
#               ABS_Y       : value 2048, min 0, max 4095
# input props:
#   INPUT_PROP_DIRECT (touchscreen)
```

---

## 9. Key Remapping trên Pi4

```
# /system/usr/keylayout/Generic.kl

key 1     ESCAPE
key 28    ENTER
key 59    F1
key 103   DPAD_UP
key 108   DPAD_DOWN

# device/brcm/rpi4/ có custom keylayout
```

```bash
# Xem keylayout
adb shell ls /system/usr/keylayout/
adb shell cat /system/usr/keylayout/Generic.kl | head -20
```

---

## 10. Input Injection (Testing)

```bash
# Inject touch qua adb (không cần root)
adb shell input tap 540 960         # Tap center
adb shell input swipe 100 500 900 500  # Swipe
adb shell input key KEYCODE_HOME    # Home button
adb shell input text "hello world"  # Type text

# Monkey testing
adb shell monkey -p com.myapp --throttle 100 10000
```

---

## 11. ANR Watchdog

```
InputDispatcher có watchdog 5 giây:
  Gửi event → App không consume trong 5s
  → ANR (Application Not Responding)

Nguyên nhân thường:
  - Main thread blocked (network, DB trên main)
  - Deadlock
  - CPU bị chiếm (GC, heavy computation)

Debug:
adb pull /data/anr/anr_*.txt
# Stack trace của tất cả threads lúc ANR
```

---

## 12. File quan trọng trong source

| File | Mô tả |
|------|-------|
| `frameworks/native/services/inputflinger/reader/InputReader.cpp` | InputReader |
| `frameworks/native/services/inputflinger/dispatcher/InputDispatcher.cpp` | InputDispatcher |
| `frameworks/native/services/inputflinger/reader/mapper/TouchInputMapper.cpp` | Touch processing |
| `kernel/common/drivers/input/touchscreen/` | Touch drivers |
| `kernel/common/drivers/hid/` | HID (USB input) |
| `kernel/common/drivers/input/evdev.c` | evdev interface |
| `system/usr/idc/` | Input device calibration |
