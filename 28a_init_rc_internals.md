# Android Init — .rc Parser và Action Engine Deep Dive

File overview `28_android_init_property_system.md` đã cover cú pháp .rc, trigger types, service directives, boot sequence tổng quan, property system. File này đào sâu vào cơ chế **bên trong** của init: parser, action queue engine, service state machine, socket creation, property shared memory.

---

## 1. .rc File Parser — Token-based Line Processing

**Source:** `system/core/init/parser.cpp`, `system/core/init/action_parser.cpp`

Parser xử lý .rc theo mô hình line-by-line tokenizer:

```cpp
// parser.cpp
void Parser::ParseConfig(const std::string& path) {
    // Đọc file thành lines
    // Mỗi line → split tokens by whitespace
    // Handle continuation lines (backslash at end)
    // Strip comments (# đến cuối dòng)
    
    for (auto& line : lines) {
        std::vector<std::string> tokens = Tokenize(line);
        if (tokens.empty()) continue;
        
        const std::string& keyword = tokens[0];
        
        if (keyword == "on") {
            // Bắt đầu Action section
            current_section_ = StartAction(tokens);
        } else if (keyword == "service") {
            // Bắt đầu Service section
            current_section_ = StartService(tokens);
        } else if (keyword == "import") {
            // Ngay lập tức import và parse file khác
            ParseConfig(ExpandProps(tokens[1]));
        } else {
            // Command trong section hiện tại
            current_section_->ParseLine(tokens);
        }
    }
}
```

**Property expansion trong import:**
```rc
import /vendor/etc/init/hw/init.${ro.hardware}.rc
# ${ro.hardware} được expand lúc parse → "rpi4" → init.rpi4.rc
# Expansion xảy ra tại parse time, không phải runtime
```

---

## 2. ActionManager — Queue Thực thi

**Source:** `system/core/init/action_manager.cpp`, `system/core/init/action.cpp`

```cpp
class ActionManager {
    std::vector<std::unique_ptr<Action>> actions_;   // Tất cả actions đã parse
    std::vector<Action*> current_executing_actions_; // Queue đang chờ execute
    Action* current_action_ = nullptr;               // Action đang execute
    
    void QueueEventTrigger(const std::string& trigger) {
        // Tìm tất cả Actions match trigger này
        for (auto& action : actions_) {
            if (action->CheckEventTrigger(trigger)) {
                current_executing_actions_.push_back(action.get());
            }
        }
    }
    
    void ExecuteOneCommand() {
        // Chỉ execute 1 command mỗi lần gọi
        if (current_executing_actions_.empty()) return;
        
        if (current_action_ == nullptr || !current_action_->HasMoreCommands()) {
            current_action_ = current_executing_actions_.front();
            current_executing_actions_.erase(current_executing_actions_.begin());
        }
        
        current_action_->ExecuteOneCommand(context_);
    }
}
```

**Tại sao chỉ execute 1 command mỗi iteration?**

Init main loop chạy `ExecuteOneCommand()` rồi lập tức xử lý I/O events (SIGCHLD, property socket). Nếu execute nhiều commands cùng lúc → init bị block trong khi service crash mà không được restart.

```
[init loop] → ExecuteOneCommand → epoll_wait → ExecuteOneCommand → ...
               (1 command)         (I/O events)   (1 command)
```

---

## 3. Trigger Types — EventTrigger vs PropertyTrigger

```cpp
class Action {
    // EventTrigger: "early-init", "boot", "shutdown"
    std::string event_trigger_;
    
    // PropertyTrigger: map của {property_name: expected_value}
    std::map<std::string, std::string> property_triggers_;
    
    bool CheckEventTrigger(const std::string& trigger) {
        return !event_trigger_.empty() && event_trigger_ == trigger;
    }
    
    bool CheckPropertyTrigger(const std::string& name, const std::string& value) {
        auto it = property_triggers_.find(name);
        if (it != property_triggers_.end()) {
            return it->second == "*" || it->second == value;
        }
        return false;
    }
    
    // Combined: "on boot && property:ro.build.type=userdebug"
    // → Chỉ execute khi BOTH conditions true
    bool CheckTriggers(/* event, props */) {
        // EventTrigger phải match
        // TẤT CẢ PropertyTriggers phải match current prop values
    }
};
```

**Property triggers fire nhiều lần:**
```rc
# File này có thể fire NHIỀU LẦN nếu property thay đổi nhiều lần
on property:sys.boot_completed=1
    start my_service
# Nếu ai đó set sys.boot_completed=0 rồi set lại =1 → fire lần 2
```

**Boot trigger order (không thể thay đổi):**
```
early-init → init → early-fs → fs → post-fs → post-fs-data →
late-fs → load_persist_props → start-zygote → boot → 
property:sys.boot_completed=1
```

---

## 4. Service State Machine

**Source:** `system/core/init/service.cpp`

```
CREATED ──────────────────────────────────────────────────►
   │                                                        │
   │ Start()                                                │ Stop()
   ▼                                                        ▼
STARTING ──fork+execve──► RUNNING ──────────────────► STOPPING
                              │                         │
                              │ process dies             │ SIGTERM → timeout → SIGKILL
                              ▼                         ▼
                          RESTARTING ◄─────────── STOPPED
                              │
                              │ restart_delay expires
                              ▼
                           RUNNING
```

```cpp
Result<void> Service::Start() {
    if (flags_ & SVC_DISABLED) return {};  // "disabled" directive
    
    // Fork
    pid_t pid = fork();
    if (pid == 0) {
        // CHILD PROCESS:
        // 1. Tạo sockets (từ "socket" directives)
        for (const auto& socket : sockets_) {
            socket.Create(context_);
        }
        
        // 2. Set scheduling policy, priority
        SetScheduler();
        
        // 3. Set supplementary groups
        setgroups(supp_gids_.size(), supp_gids_.data());
        
        // 4. Set GID, UID (SAU setgroups vì sau setuid không thể setgroups)
        setgid(gid_);
        setuid(uid_);
        
        // 5. Set capabilities
        SetCapabilities();
        
        // 6. Set SELinux label
        setexeccon(seclabel_.c_str());
        
        // 7. execve
        execve(args_[0].c_str(), argv, envp);
        _exit(127);  // execve failed
    }
    
    // PARENT (init):
    pid_ = pid;
    time_started_ = boot_clock::now();
    flags_ |= SVC_RUNNING;
    
    // Write PID vào writepid files (ví dụ: /dev/cpuset/tasks)
    WritePidToFiles();
}
```

---

## 5. Restart Policy — Backoff và Critical

```cpp
void Service::Reap(const siginfo_t& siginfo) {
    // Process vừa die
    flags_ &= ~SVC_RUNNING;
    
    if (flags_ & SVC_ONESHOT) {
        // "oneshot" → không restart
        return;
    }
    
    // Track crash history
    time_t now = time(nullptr);
    crash_times_.push_back(now);
    
    // Xóa crashes cũ hơn 4 phút
    crash_times_.erase(
        std::remove_if(crash_times_.begin(), crash_times_.end(),
                       [now](time_t t) { return now - t > 240; }),
        crash_times_.end());
    
    if (crash_times_.size() >= 4) {
        // 4 crashes trong 4 phút
        if (flags_ & SVC_CRITICAL) {
            // "critical" → reboot to recovery
            InitFatalReboot(SIGABRT);
        } else {
            // Normal service → disable, log warning
            flags_ |= SVC_DISABLED;
            LOG(ERROR) << "Service '" << name_ << "' crashed 4 times, disabled";
            return;
        }
    }
    
    // Schedule restart sau kMinCrashPeriod (4 seconds)
    flags_ |= SVC_RESTARTING;
    time_started_ = {};
    // Main loop: RestartProcesses() kiểm tra và restart sau delay
}
```

---

## 6. Socket Creation — Trước khi fork

Init tạo socket TRƯỚC khi fork, truyền FD vào child qua environment:

```cpp
// ServiceSocket::Create()
void ServiceSocket::Create(const std::string& context) {
    int fd = -1;
    
    if (type_ == "stream") {
        fd = socket(AF_UNIX, SOCK_STREAM | SOCK_CLOEXEC | SOCK_NONBLOCK, 0);
    } else if (type_ == "dgram") {
        fd = socket(AF_UNIX, SOCK_DGRAM | SOCK_CLOEXEC, 0);
    } else if (type_ == "seqpacket") {
        fd = socket(AF_UNIX, SOCK_SEQPACKET | SOCK_CLOEXEC, 0);
    }
    
    // Bind tới /dev/socket/<name>
    struct sockaddr_un addr;
    snprintf(addr.sun_path, sizeof(addr.sun_path), "/dev/socket/%s", name_.c_str());
    bind(fd, (sockaddr*)&addr, sizeof(addr));
    
    // Set SELinux label
    fsetxattr(fd, XATTR_NAME_SELINUX, context.c_str(), ...);
    
    // Chmod, chown
    fchown(fd, uid_, gid_);
    
    // Lưu FD vào environment cho child
    // Khi fork+execve, child inherit FD này
    // ANDROID_SOCKET_<name>=<fd>
    fcntl(fd, F_SETFD, 0);  // Clear CLOEXEC để child inherit
    setenv(("ANDROID_SOCKET_" + name_).c_str(),
           std::to_string(fd).c_str(), 1);
}

// Child process gọi:
int fd = android_get_control_socket("zygote");
// → getenv("ANDROID_SOCKET_zygote") → parse int → return fd
```

**Tại sao init tạo socket trước, không phải service tự tạo?**

Với supervisor model, service có thể crash và restart. Socket phải tồn tại LIÊN TỤC (clients connect ngay cả khi service đang restart). Init giữ socket FD, service chỉ inherit — nếu service die, socket vẫn bound, client không bị disconnect.

---

## 7. Property Shared Memory Protocol

**Source:** `system/core/init/property_service.cpp`, `bionic/libc/bionic/system_property_api.cpp`

```
/dev/__properties__/
├── property_info          ← Trie structure (property name → context)
├── u:object_r:build_prop:s0    ← Shared mem file cho build_prop context
├── u:object_r:system_prop:s0   ← Shared mem cho system_prop
└── u:object_r:vendor_prop:s0   ← Shared mem cho vendor_prop
```

**Structure trong shared memory:**

```c
// bionic/libc/private/system_properties/prop_area.h
struct prop_area {
    uint32_t bytes_used;
    atomic_uint_least32_t serial;  // tăng mỗi khi property thay đổi
    uint32_t magic;
    uint32_t version;
    uint32_t reserved[28];
    char data[0];  // Variable length: trie nodes + prop_info objects
};

struct prop_info {
    atomic_uint_least32_t serial;  // version của property này
    union {
        char value[PROP_VALUE_MAX];     // <= 92 bytes
        struct {
            char error_message[56];
            uint32_t offset;            // offset đến long value (>92 bytes)
        };
    };
    char name[0];  // property name, null-terminated
};
```

**Read path — ZERO IPC:**
```c
// __system_property_get() — không cần IPC!
// 1. mmap /dev/__properties__/<context_file> nếu chưa map
// 2. Walk trie với property name → tìm prop_info
// 3. Đọc prop_info.value trực tiếp từ shared memory
// Tất cả processes đều mmap cùng physical pages → đọc trực tiếp
```

**Write path — qua Unix socket:**
```c
// property_set() → socket → init:
// 1. Client mở /dev/socket/property_service
// 2. Gửi: {name_len, name, value_len, value}
// 3. Init nhận, validate (SELinux check), update prop_info.value
// 4. Atomic increment serial
// 5. Close socket
```

**Wait for property — polling với futex:**
```c
// __system_property_wait() — efficient wait
// Dùng serial number: sleep nếu serial không thay đổi
// Kernel futex: không busy-wait
uint32_t serial = __system_property_serial(pi);
while (__system_property_serial(pi) == serial) {
    // futex_wait trên pi->serial
    __futex_wait(&pi->serial, serial, timeout);
}
// Serial thay đổi → property đã được set
```

---

## 8. Property Trigger Sequencing

```
Khi property thay đổi:
  1. init nhận trên property_service socket
  2. ValidateProperty(): SELinux check (property_contexts)
  3. WriteToSystemProperty(): update shared memory
  4. PropertyChanged(name, value):
     → ActionManager::QueuePropertyTrigger(name, value)
     → Scan tất cả actions → add matching vào queue
  5. Main loop: execute queued actions
```

**Timing subtlety — property trigger trước property visible:**

```
t=0: sys.boot_completed không tồn tại
t=1: Zygote viết sys.boot_completed=1
t=2: Property service update shared memory
t=3: Init trigger action "on property:sys.boot_completed=1"
t=4: Action execute (có thể delay nếu queue dài)

→ Khoảng thời gian [t=2, t=4]: sys.boot_completed=1 nhưng action chưa execute
→ Apps đọc property thấy =1 nhưng late_service chưa chạy
```

---

## 9. ueventd — Device Node Creation Engine

**Source:** `system/core/init/ueventd.cpp`

```cpp
void UeventdMain() {
    // Mở NETLINK_KOBJECT_UEVENT socket
    int nl_fd = socket(PF_NETLINK, SOCK_DGRAM | SOCK_CLOEXEC,
                        NETLINK_KOBJECT_UEVENT);
    bind(nl_fd, (sockaddr*)&addr, sizeof(addr));
    
    // Trigger coldplug: đọc sự kiện cũ từ /sys
    ColdBoot(nl_fd);
    
    // Main loop: handle hotplug events
    while (true) {
        Uevent uevent;
        ReadUevent(nl_fd, &uevent);
        HandleUevent(uevent);
    }
}

void HandleUevent(const Uevent& uevent) {
    // Parse: ACTION, DEVPATH, SUBSYSTEM, DEVNAME, DEVTYPE
    if (uevent.action == "add") {
        MakeDevice(uevent.devname, uevent.devpath, ...);
    } else if (uevent.action == "remove") {
        unlink(("/dev/" + uevent.devname).c_str());
    }
}

void MakeDevice(const std::string& devname, ...) {
    // 1. Lookup permissions từ ueventd.rc
    // /dev/gpiochip* → 0660, root, gpio
    Permissions perms = FindPermissions(devname);
    
    // 2. mknod với major:minor từ uevent
    mknod(path.c_str(), mode | perms.perm, makedev(major, minor));
    
    // 3. chown + chmod
    chown(path.c_str(), perms.uid, perms.gid);
    
    // 4. SELinux label từ file_contexts
    std::string context = GetFileContext(path);
    lsetfilecon(path.c_str(), context.c_str());
}
```

**Pi4 GPIO/I2C uevent sequence:**
```
Kernel boot → probe bcm2835_gpio driver
  → uevent: ACTION=add DEVPATH=/class/gpio DEVNAME=gpiochip0
  → ueventd: mknod /dev/gpiochip0, chown root:gpio, chmod 0660
  → uevent: ACTION=add DEVPATH=/bus/i2c/devices/i2c-1 DEVNAME=i2c-1
  → ueventd: mknod /dev/i2c-1, chown root:i2c, chmod 0660
```

---

## 10. Init Main Loop — epoll + Action Execution

**Source:** `system/core/init/init.cpp`

```cpp
int main(int argc, char** argv) {
    // ... setup ...
    
    Epoll epoll;
    
    // Register event sources
    epoll.RegisterHandler(signal_fd, HandleSignalFd);           // SIGCHLD
    epoll.RegisterHandler(property_fd, HandlePropertyFd);       // Property requests
    epoll.RegisterHandler(keychord_fd, HandleKeychordFd);       // Debug keychords
    
    while (true) {
        // Execute pending action command (nếu không đang chờ exec service)
        if (!(waiting_for_prop || Service::is_exec_service_running())) {
            am.ExecuteOneCommand();
        }
        
        // Restart crashed services
        if (!IsShuttingDown()) {
            RestartProcesses();
        }
        
        // Decide epoll timeout
        auto next_process_restart_time = RestartProcessesIfNeeded();
        
        int epoll_timeout_ms = -1;  // block indefinitely by default
        
        if (am.HasMoreCommands()) {
            epoll_timeout_ms = 0;   // return immediately (don't block)
        }
        if (next_process_restart_time) {
            epoll_timeout_ms = /* time until next restart */;
        }
        
        // Wait for I/O events (với timeout)
        auto pending_functions = epoll.Wait(epoll_timeout_ms);
        
        for (const auto& f : pending_functions) {
            (*f)();  // HandleSignalFd, HandlePropertyFd, ...
        }
    }
}
```

**HandleSignalFd — SIGCHLD xử lý:**
```cpp
void HandleSignalFd() {
    // Đọc signal info từ signalfd
    signalfd_siginfo siginfo;
    read(signal_fd, &siginfo, sizeof(siginfo));
    
    if (siginfo.ssi_signo == SIGCHLD) {
        // Reap zombie processes
        pid_t pid;
        while ((pid = waitpid(-1, &status, WNOHANG)) > 0) {
            // Tìm Service với pid này
            Service* svc = ServiceList::GetInstance().FindService(pid, &Service::pid_);
            if (svc) {
                svc->Reap(siginfo);  // → restart logic
            }
        }
    }
}
```

**exec_start vs start:**
```rc
# "start" là async → init không chờ
start vold

# "exec_start" là synchronous → init chờ process exit
exec_start apexd
# init bị block: is_exec_service_running() = true
# → ExecuteOneCommand() không chạy cho đến khi apexd exit
# Dùng cho: các script setup cần chạy xong trước khi tiếp tục boot
```
