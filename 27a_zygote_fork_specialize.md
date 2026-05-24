# Zygote fork-and-specialize — Deep Dive

Zygote là "mother process" của Android: tất cả app process đều được fork từ đây. Cơ chế này giúp app launch nhanh (không phải load ART từ đầu) nhưng đòi hỏi một protocol phức tạp để đảm bảo isolation đúng sau fork.

---

## 1. Kiến trúc tổng thể

```
ActivityManagerService (system_server)
  │  LocalSocket ("zygote" / "zygote_secondary")
  │  writeInt(numArgs) + args...
  ▼
ZygoteServer.runSelectLoop()          ← main loop Zygote process
  │
  ├── ZygoteConnection.processCommand()
  │     └── Zygote.forkAndSpecialize()
  │           └── nativeForkAndSpecialize() [JNI]
  │                 └── ForkCommon() → fork()
  │                       ├── Child:  SpecializeCommon()
  │                       └── Parent: handleParentProc()
  │
  └── USAP pool (pre-forked processes)
        └── ZygoteServer.attemptUsapSendArgsAndGetResult()
```

---

## 2. ZygoteServer.runSelectLoop() — Event Loop

```java
// frameworks/base/core/java/com/android/internal/os/ZygoteServer.java

Runnable runSelectLoop(String abiList) {
    ArrayList<FileDescriptor> socketFDs = new ArrayList<>();
    ArrayList<ZygoteConnection> peers = new ArrayList<>();

    // FD 0: server socket (accept new connections)
    socketFDs.add(mZygoteSocket.getFileDescriptor());

    while (true) {
        // Build pollFDs array từ socketFDs + USAP FDs
        StructPollfd[] pollFDs = new StructPollfd[socketFDs.size()];
        for (int i = 0; i < socketFDs.size(); i++) {
            pollFDs[i] = new StructPollfd();
            pollFDs[i].fd = socketFDs.get(i);
            pollFDs[i].events = (short) POLLIN;
        }

        // Block on poll() — chờ input từ bất kỳ FD nào
        // pollTimeoutMs = -1: block forever (hoặc timeout ngắn nếu có USAP work)
        int pollReturnValue = Os.poll(pollFDs, pollTimeoutMs);

        // Xử lý từng FD có activity
        for (int i = pollFDs.length - 1; i >= 0; --i) {
            if ((pollFDs[i].revents & POLLIN) == 0) continue;

            if (i == 0) {
                // Server socket: accept new connection từ AMS
                ZygoteConnection newPeer = acceptCommandPeer(abiList);
                peers.add(newPeer);
                socketFDs.add(newPeer.getFileDescriptor());
            } else if (i < usapPoolFDIndex) {
                // Existing connection: process command (fork request)
                ZygoteConnection connection = peers.get(i);
                final Runnable command = connection.processCommand(this, multipleForksOK);

                // Nếu đây là child process sau fork:
                if (mIsForkChild) {
                    // Child: return runnable để execute app's main()
                    // KHÔNG tiếp tục loop (child thoát khỏi runSelectLoop)
                    return command;
                }
                // Parent: command == null, tiếp tục loop
            } else {
                // USAP pool FD: child đã được specialize
                // Đọc kết quả specialization từ USAP child
                ZygoteServer.removeUsapTableEntry(i - usapPoolFDIndex);
            }
        }

        // Refill USAP pool nếu cần
        if (mUsapPoolRefillAction != USAP_POOL_REFILL_DISABLE) {
            refillUsapPool();
        }
    }
}
```

**Tại sao iterate ngược (i-- from end)?**

Khi remove FD khỏi list, index của các FDs phía sau thay đổi. Iterate ngược đảm bảo remove an toàn mà không skip entry.

---

## 3. ZygoteConnection.processCommand() — Parse Fork Request

### 3.1 Wire Protocol

```
AMS → Zygote socket (text-based, line per argument):
┌────────────────────────────────────────────────┐
│  "7\n"                    ← số arguments       │
│  "--uid=10053\n"                                │
│  "--gid=10053\n"                                │
│  "--gids=1003,1015,3003\n" ← supplementary GIDs│
│  "--target-sdk-version=34\n"                    │
│  "--nice-name=com.example.app\n"                │
│  "--runtime-args\n"                             │
│  "com.example.app.MainActivity\n" ← main class  │
└────────────────────────────────────────────────┘
```

### 3.2 Argument Parsing

```java
// ZygoteArguments.java — container cho tất cả fork params
class ZygoteArguments {
    int     mUid;
    int     mGid;
    int[]   mGids;                // supplementary groups
    int     mTargetSdkVersion;
    String  mNiceName;            // process name
    String  mInstructionSet;      // arm64, x86_64
    boolean mEnableDebugger;
    boolean mEnablePtrace;
    boolean mMountExternal;       // mount storage namespace
    String  mSeInfo;              // SELinux info
    String[] mRemainingArgs;      // class name + args

    // Capabilities
    long    mPermittedCapabilities;
    long    mEffectiveCapabilities;
    long    mInheritableCapabilities;
}

// processCommand() flow:
Runnable processCommand(ZygoteServer server, boolean multipleOK) {
    // 1. Đọc args từ socket
    String[] args = Zygote.readArgumentList(mReader);

    // 2. Parse vào ZygoteArguments
    ZygoteArguments parsedArgs = ZygoteArguments.getInstance(args);

    // 3. Validate: uid/gid phải được AMS set, không phải app tự set
    applyUidSecurityPolicy(parsedArgs, peer);
    applyCapabilitiesSecurityPolicy(parsedArgs, ...);

    // 4. Fork!
    pid = Zygote.forkAndSpecialize(
            parsedArgs.mUid, parsedArgs.mGid, parsedArgs.mGids,
            parsedArgs.mRuntimeFlags, rlimits,
            parsedArgs.mMountExternal,
            parsedArgs.mSeInfo, parsedArgs.mNiceName,
            fdsToClose, fdsToIgnore,
            parsedArgs.mStartChildZygote,
            parsedArgs.mInstructionSet,
            parsedArgs.mAppDataDir,
            parsedArgs.mIsTopApp,
            parsedArgs.mPkgDataInfoList,
            parsedArgs.mAllowlistedDataInfoList,
            parsedArgs.mBindMountAppDataDirs,
            parsedArgs.mBindMountAppStorageDirs);

    if (pid == 0) {
        // Child process
        server.setForkChild();  // set mIsForkChild = true
        return handleChildProc(parsedArgs, descriptors, childPipeFd, ...);
    } else {
        // Parent (Zygote)
        handleParentProc(pid, serverPipeFd);
        return null;
    }
}
```

---

## 4. Zygote.forkAndSpecialize() — JNI Bridge

```java
// frameworks/base/core/java/com/android/internal/os/Zygote.java

static int forkAndSpecialize(int uid, int gid, int[] gids,
        int runtimeFlags, ...) {

    // Pre-fork: flush ART internals
    ZygoteHooks.preFork();
    // preFork() làm gì:
    //   - stopSafePointHandling(): pause GC và JIT
    //   - deinitializeMainThreadForFork(): xóa thread-local state
    //   → Đảm bảo fork() không copy GC state không nhất quán

    int pid = nativeForkAndSpecialize(uid, gid, gids, runtimeFlags, ...);

    // Post-fork (chạy ở CẢ parent lẫn child)
    ZygoteHooks.postForkCommon();
    // postForkCommon():
    //   - startJitCompilationThread(): restart JIT thread
    //   - Chỉ child: reinitialize runtime state

    return pid;
}
```

---

## 5. nativeForkAndSpecialize() — C++ JNI Implementation

```cpp
// frameworks/base/core/jni/com_android_internal_os_Zygote.cpp

static jint com_android_internal_os_Zygote_nativeForkAndSpecialize(
        JNIEnv* env, jclass, jint uid, jint gid, jintArray gids,
        jint runtime_flags, ...)
{
    // Collect FDs that should NOT be closed in child
    // (kZygoteChildFileDescriptors: stdin/stdout/stderr + some system FDs)
    std::vector<int> fds_to_close(fdsToClose.begin(), fdsToClose.end());
    std::vector<int> fds_to_ignore(fdsToIgnore.begin(), fdsToIgnore.end());

    // Actual fork
    pid_t pid = ForkCommon(env, /* is_system_server */ false,
                            fds_to_close, fds_to_ignore,
                            /* is_priority_fork */ true);

    if (pid == 0) {
        // Child: specialize
        SpecializeCommon(env, uid, gid, gids, runtime_flags,
                         permitted_capabilities, effective_capabilities,
                         mount_external, se_info, nice_name,
                         false, /* is_child_zygote */ false,
                         instruction_set, app_data_dir,
                         is_top_app, pk_data_info_list,
                         allowlisted_data_info_list, ...);
    }
    return pid;
}
```

### 5.1 ForkCommon() — Actual fork()

```cpp
static pid_t ForkCommon(JNIEnv* env, bool is_system_server,
                         const std::vector<int>& fds_to_close,
                         const std::vector<int>& fds_to_ignore,
                         bool is_priority_fork)
{
    // 1. Setup SIGCHLD handler (để Zygote track child deaths)
    SetSignalHandlers();

    // 2. Block signals trong Zygote trước khi fork
    //    → Đảm bảo child không nhận signal ngay sau fork
    BlockSignal(SIGCHLD, fail_fn);
    BlockSignal(SIGHUP, fail_fn);

    // 3. flush stdio
    __android_log_close();

    // 4. fork()!
    pid_t pid = fork();
    // Kernel: copy_process()
    //   - dup_mm(): copy memory descriptors (COW page tables)
    //   - dup_fd(): copy file descriptor table
    //   - copy_sighand(): copy signal handlers
    //   - copy_signal(): copy signal state
    //   - thread group: child là single-threaded (chỉ forking thread)

    if (pid == 0) {
        // Child process only:

        // 5. Unblock signals
        UnblockSignal(SIGCHLD, fail_fn);

        // 6. Đóng FDs không cần thiết
        //    (tất cả Zygote's sockets, server FDs)
        DetachDescriptors(env, fds_to_close, fail_fn);

        // 7. Tạo new session (setsid())
        //    → Child không còn controlling terminal
        setsid();

    } else {
        // Parent (Zygote):
        UnblockSignal(SIGCHLD, fail_fn);
        UnblockSignal(SIGHUP, fail_fn);
    }

    return pid;
}
```

---

## 6. fork() Kernel Behavior — What Kernel Does

```
Parent process (Zygote)                    Child process (new app)
─────────────────────────────────────────────────────────────────
mm_struct (page tables)                     mm_struct (copy)
  │ pte entries → physical pages              │ pte entries → SAME physical pages
  │ all marked READ-ONLY                      │ all marked READ-ONLY
  └─ On write: page fault → COW copy         └─ On write: page fault → COW copy

fd_struct                                   fd_struct (copy)
  │ stdin/stdout/stderr → /dev/null           │ → same files (ref counted)
  │ Zygote sockets → (will be closed)         │ (closed via DetachDescriptors)
  └─ shared libs, etc.                        └─ shared libs same physical pages

signal_struct                               signal_struct
  └─ handlers copied                         └─ pending signals cleared

threads                                     threads
  └─ ALL threads (1000+)                     └─ ONLY forking thread
       WARNING: other threads gone!               → must reinitialize mutexes,
                                                    locks, condition vars
```

**Critical post-fork concern:** Zygote có nhiều threads (GC threads, JIT threads, binder threads). Sau fork(), child chỉ có 1 thread. Nếu Zygote thread đang hold mutex khi fork xảy ra → child deadlocks khi cố acquire mutex đó. Đây là lý do `preFork()` dừng GC và JIT trước fork.

---

## 7. SpecializeCommon() — Child Specialization (15 bước)

```cpp
static void SpecializeCommon(JNIEnv* env, uid_t uid, gid_t gid,
        jintArray javaGids, jint runtime_flags,
        jlong permitted_capabilities, jlong effective_capabilities,
        jint mount_external, jstring se_info, jstring nice_name, ...)
{
    // Thứ tự CỰC KỲ quan trọng — không được đổi

    // 1. Mount storage namespace (trước khi drop privileges)
    //    → Cần root để bind mount
    if (mount_external != IVold::REMOUNT_MODE_NONE) {
        MountEmulatedStorage(uid, mount_external, fail_fn);
        // bind mounts /storage/emulated/<userId> vào /storage/self/primary
        // Mỗi app có view riêng của external storage
    }

    // 2. Drop bounding set (capabilities mà root không thể re-acquire)
    DropCapabilitiesBoundingSet(fail_fn);

    // 3. Set supplementary groups
    SetGids(env, javaGids, is_child_zygote, fail_fn);
    // setgroups(N, gids): app có thể access /sdcard, Bluetooth, etc.

    // 4. Set resource limits
    SetRLimits(env, javaRlimits, fail_fn);
    // setrlimit(RLIMIT_NOFILE, ...), etc.

    // 5. Set scheduler policy (nếu cần)
    if (setSchedPolicy) {
        SchedPriorityRestoreGuard priorityGuard{};
        SetSchedulerPolicy(fail_fn);
    }

    // 6. Set personality (process execution domain)
    if (runtime_flags & RuntimeFlags::ONLY_USE_SYSTEM_OAT_FILES) {
        SetArtDex2OatFlag(fail_fn);
    }

    // 7. Set uid/gid (PHẢI SAU mount, TRƯỚC capabilities)
    if (setresuid(uid, uid, uid) == -1) {
        fail_fn(CREATE_ERROR("setresuid(%d) failed", uid));
    }
    if (setresgid(gid, gid, gid) == -1) {
        fail_fn(CREATE_ERROR("setresgid(%d) failed", gid));
    }
    // Từ đây: process chạy với app UID, KHÔNG phải root
    // Không thể làm gì cần root nữa (trừ capabilities được giữ)

    // 8. Set capabilities (permitted + effective)
    //    Capabilities là subset root đã cho phép
    //    Ví dụ: WAKE_LOCK cho foreground services
    SetCapabilities(permitted_capabilities, effective_capabilities,
                    permitted_capabilities, fail_fn);

    // 9. Set thread priority (nice value)
    if (setpriority(PRIO_PROCESS, 0, PROCESS_PRIORITY_DEFAULT) != 0) {
        ALOGW("setpriority failed");
    }

    // 10. Install seccomp filter (PHẢI sau set uid/gid)
    //     Seccomp filter blocks syscalls nguy hiểm
    //     Sau khi install, không thể gọi setuid/setgid nữa!
    SetUpSeccompFilter(uid, is_child_zygote, fail_fn);

    // 11. Set SELinux context (process context)
    //     Ví dụ: "u:r:untrusted_app:s0:c512,c768"
    if (!is_system_server) {
        std::string context = GetProcessContext(se_info);
        if (selinux_android_setcon(context.c_str()) < 0) {
            fail_fn("selinux_android_setcon failed");
        }
    }

    // 12. Set process name
    if (nice_name != nullptr) {
        prctl(PR_SET_NAME, nice_name);  // /proc/self/comm
    }

    // 13. ART runtime re-initialization cho child
    //     Restart GC daemon, reinitialize heap, etc.
    env->CallStaticVoidMethod(gZygoteClass, gCallPostForkChildHooks,
                              runtime_flags, is_system_server,
                              is_child_zygote, instructionSet);
    // → ZygoteHooks.postForkChild():
    //   - Runtime.getInstance().initAfterFork()
    //   - Restart heap GC threads
    //   - Reinitialize JDWP debug agent (nếu debuggable)

    // 14. Enable memory tagging (nếu được config)
    if (runtime_flags & RuntimeFlags::MEMORY_TAG_LEVEL_MASK) {
        // ARM MTE: hardware memory tagging
        HeapTaggingLevel level = ...;
        mallopt(M_HEAP_TAGGING_LEVEL, level);
    }

    // 15. Notify system về new process
    android::procinfo::SetProcessProfile(uid, gid, ...);
}
```

---

## 8. Post-Fork Parent Side

```cpp
// handleParentProc() trong ZygoteConnection.java

void handleParentProc(int pid, FileDescriptor serverPipeFd) {
    // Ghi pid của child về cho AMS qua socket
    // AMS dùng pid để track process lifecycle

    if (pid > 0) {
        try {
            // Write response: "pid\n" ← AMS đọc này
            mSocketOutStream.writeInt(pid);
            mSocketOutStream.writeBoolean(usingWrapper);
        } catch (IOException ex) {
            // AMS đã close connection (timeout?)
        }
    }
}

// SigChldHandler trong C++: xử lý SIGCHLD khi child die
static void SigChldHandler(int /*signal_number*/, siginfo_t* info,
                             void* /*ucontext*/) {
    pid_t pid;
    int status;

    // Non-blocking waitpid để collect zombie processes
    while ((pid = waitpid(-1, &status, WNOHANG)) > 0) {
        // Log child death
        if (WIFEXITED(status)) {
            ALOGI("Process %d exited cleanly (%d)", pid, WEXITSTATUS(status));
        } else if (WIFSIGNALED(status)) {
            ALOGI("Process %d exited due to signal %d", pid, WTERMSIG(status));
        }
        // AMS sẽ tự detect qua BinderDied hoặc polling /proc/pid
    }
}
```

---

## 9. handleChildProc() → App main()

```java
// Child sau SpecializeCommon(), tiếp tục trong Java:
private Runnable handleChildProc(ZygoteArguments parsedArgs, ...) {

    // Đóng socket connection tới Zygote server
    // (child không phải server, không cần listen)
    closeSocket();

    // Set process name trong ActivityThread
    android.os.Process.setArgV0(parsedArgs.mNiceName);

    if (parsedArgs.mInvokeWith != null) {
        // Debug wrapper (như valgrind, strace): exec new process
        WrapperInit.execApplication(parsedArgs.mInvokeWith, ...);
    } else {
        // Normal: RuntimeInit.applicationInit()
        return ZygoteInit.zygoteInit(
                parsedArgs.mTargetSdkVersion,
                parsedArgs.mDisabledCompatChanges,
                parsedArgs.mRemainingArgs,
                null /* classLoader */);
    }
}

// ZygoteInit.zygoteInit():
static Runnable zygoteInit(int targetSdkVersion, long[] disabledCompatChanges,
                             String[] argv, ClassLoader classLoader) {
    RuntimeInit.commonInit();   // timezone, default exception handler, etc.
    ZygoteInit.nativeZygoteInit();  // start Binder thread pool!
    return RuntimeInit.applicationInit(targetSdkVersion,
                                       disabledCompatChanges, argv, classLoader);
}

// RuntimeInit.applicationInit():
protected static Runnable applicationInit(int targetSdkVersion, ...,
                                           String[] argv, ...) {
    // argv[0] = "android.app.ActivityThread"
    // Invoke main() via reflection
    return findStaticMain(args.startClass, args.startArgs, classLoader);
    // → ActivityThread.main() → Looper.loop() → app runs!
}
```

---

## 10. USAP Pool — Pre-Forked Processes

### 10.1 Motivation

```
Cold fork path (không có USAP):
  AMS request → Zygote forks → child specializes → app starts
  Time: ~100-200ms (fork + ART init thêm 50ms)

USAP path (Unspecialized App Process):
  [background] Zygote pre-forks N processes, each waits on socket
  [app launch] AMS sends args to nearest USAP → specializes → starts
  Time: ~50-100ms (no fork overhead, just specialize)
```

### 10.2 USAP Fork và Wait Loop

```java
// Zygote.forkUsap():
static int forkUsap(LocalServerSocket usapPoolSocket, ...) {
    // Tạo socketpair để communicate với USAP child
    int[] sessionSocketRawFDs = sessionSocket.getFileDescriptors();

    pid_t pid = Zygote.forkUsap(usapPoolSocket, sessionSocketRawFDs, ...);
    // → nativeForkUsap() → ForkCommon() → fork()

    if (pid == 0) {
        // USAP child: chờ specialization command
        childMain(usapPoolSocket, ...);
        // Không return — block trong childMain()
    }
    return pid;
}

// childMain(): USAP child loop
static Runnable childMain(LocalSocket usapSocket, ...) {
    // Loop chờ connection từ AMS (qua ZygoteServer USAP path)
    while (true) {
        Os.poll(new StructPollfd[]{...}, -1);  // block

        // Nhận specialization args từ AMS
        ZygoteArguments args = ZygoteArguments.getInstance(
                                    Zygote.readArgumentList(reader));

        // Specialize ngay lập tức (không cần fork thêm)
        SpecializeCommon(env, args.mUid, args.mGid, ...);

        // Execute app main()
        return handleChildProc(args, ...);
    }
}
```

### 10.3 USAP Pool Management

```java
// ZygoteServer.java
private static final int sUsapPoolSizeMin = 1;
private static final int sUsapPoolSizeMax = 10;  // từ ZygoteConfig
private static final int sUsapPoolRefillThreshold = 1;

void refillUsapPool() {
    int currentSize = Zygote.getUsapPoolCount();
    int targetSize = Zygote.getUsapPoolEventFDCount();

    // Fork thêm USAP processes nếu pool còn ít
    while (currentSize < targetSize) {
        Zygote.forkUsap(mUsapPoolSocket, mUsapPoolSocketFDs, ...);
        currentSize++;
    }
}

// Triggers cho refill:
// 1. Sau khi USAP bị dùng (pool size giảm)
// 2. System property change: persist.device_config.runtime_native.usap_pool_size
// 3. Sau boot complete
```

### 10.4 USAP vs Cold Fork — Tradeoff

```
USAP advantages:
  - Không cần fork() tại thời điểm launch → faster
  - No copy_process() overhead (page table copy)
  - Child đã warm up (ART classes loaded)

USAP disadvantages:
  - Memory overhead: N pre-forked processes idle
  - USAP child chưa biết sẽ specialize thành app gì
  → Không thể pre-load app-specific code
  → Chỉ share generic Zygote heap, không share app-specific

Khi nào cold fork được dùng thay USAP:
  - USAP pool empty
  - App needs special flags (USAP không support hết)
  - is_child_zygote = true (App Zygote như WebView process)
```

---

## 11. seccomp Filter

### 11.1 Timing

```cpp
// SetUpSeccompFilter() được gọi SAU setresuid() — lý do:
//
// seccomp filter có thể block syscall setuid/setgid
// Nếu install seccomp TRƯỚC setresuid():
//   → Child không thể set uid/gid → security failure
//
// Thứ tự đúng:
//   setresuid/setresgid → install seccomp → (từ đây không gọi được setuid)

static void SetUpSeccompFilter(uid_t uid, bool is_child_zygote,
                                 fail_fn_t fail_fn) {
    if (!gIsSeccompEnforced) return;

    // App processes: full restrictive filter
    // Child zygote (WebView): different filter
    if (is_child_zygote) {
        set_app_zygote_seccomp_filter();
    } else {
        set_app_seccomp_filter();
        // Blocks: kexec_load, ptrace (attach), perf_event_open,
        //         bpf, userfaultfd, io_uring (security sensitive)
    }
}
```

### 11.2 Syscalls bị block cho app

```
Blocked:
  ptrace (PTRACE_ATTACH)  → ngăn app debug process khác
  perf_event_open         → ngăn side-channel attacks
  bpf                     → ngăn load eBPF programs
  kexec_load              → ngăn replace kernel
  io_uring                → attack surface reduction (Android 12+)
  
Allowed (vì app cần):
  read/write/open/close
  mmap/mprotect/mlock
  clone (threads), futex
  socket/connect (networking)
  ioctl (graphics, audio)
  ...tất cả syscalls thông thường
```

---

## 12. COW Memory Behavior Sau Fork

```bash
# Sau fork, xem RSS và PSS của Zygote vs app process:
adb shell cat /proc/$(pidof zygote)/smaps | grep -A5 "dalvik-zygote"
# Rss: 50000 kB   ← Resident (physical pages)
# Pss:  5000 kB   ← Proportional share (shared / num_processes)

# App process:
adb shell cat /proc/$(pidof com.example.app)/smaps | grep -E "(Rss|Pss)" | head

# COW examples:
# /system/framework/arm64/boot.oat:   PSS ≈ RSS/N (shared bởi N apps)
# heap (app-specific):                PSS ≈ RSS (private, không shared)
# stack:                              PSS ≈ RSS (private)
```

```
Pages shared với Zygote (read-only → không trigger COW):
  - /system/framework/*.oat (ART pre-compiled code)
  - /system/lib64/*.so (system libraries)
  - /data/dalvik-cache/*.odex (optimized dex)
  - ART internal structures (intern table, class table)

Pages private sau COW (write triggers copy):
  - App heap (new objects)
  - Zygote heap dirty pages (GC có thể dirty pages)
  - Thread stacks
  - Process-specific metadata
```

---

## 13. App Zygote và WebView Zygote

```
System Zygote (zygote64)
  ├── All regular apps
  └── App Zygote (per-app, isolated)
        └── WebView Zygote (webview_zygote)
              └── WebView render processes
                  (isolated from app, additional sandbox)

WebView Zygote:
  - Là child của system Zygote, fork xảy ra lúc boot
  - Loads com.android.webview library vào process
  - Khi app dùng WebView: system Zygote KHÔNG fork trực tiếp
    → App Zygote fork từ WebView Zygote
  - Chia sẻ WebView library pages với tất cả WebView processes
  - Strong isolation: SECCOMP renderer filter (stricter than app)
```

---

## 14. Debug Commands

```bash
# Xem Zygote socket
adb shell ls -la /dev/socket/zygote*
# srw-rw---- zygote wifi     /dev/socket/zygote
# srw-rw---- zygote wifi     /dev/socket/zygote_secondary (32-bit)

# Xem USAP pool
adb shell getprop persist.device_config.runtime_native.usap_pool_enabled
# true/false

adb shell cat /proc/$(pidof zygote)/status | grep -E "(Pid|Threads|VmRSS|VmPSS)"

# Xem process tree
adb shell ps -ef | grep zygote
# root      571    1  0 ...  zygote64
# root      572    1  0 ...  zygote         (32-bit Zygote)

# Xem fork latency với systrace
adb shell atrace --async_start -b 32768 -c activity am
# → systrace sẽ show:
#   "Zygote:ForkAndSpecialize" mark
#   duration thường 20-80ms

# Xem capabilities của app process
adb shell cat /proc/$(pidof com.example.app)/status | grep Cap
# CapPrm: 0000000000000000  (no permitted caps for regular app)
# CapEff: 0000000000000000
# CapBnd: 0000003fffffffff

# Xem seccomp status
adb shell cat /proc/$(pidof com.example.app)/status | grep Seccomp
# Seccomp: 2   (2 = SECCOMP_MODE_FILTER)

# Xem SELinux context
adb shell cat /proc/$(pidof com.example.app)/attr/current
# u:r:untrusted_app:s0:c512,c768

# Theo dõi fork trong strace
adb shell strace -e trace=clone,fork -p $(pidof zygote64)
```

---

## 15. Full Sequence: App Launch với USAP

```
ActivityManagerService                Zygote                    USAP Child
───────────────────────────────────────────────────────────────────────────
                         [boot time: refillUsapPool()]
                           fork() → USAP child #1
                           fork() → USAP child #2
                                                         childMain():
                                                           poll() [blocking]

startProcess("com.example.app")
  LocalSocket.connect("zygote")
  write args (uid, gid, etc.)  ──────────────────────►
                               runSelectLoop():
                               poll() returns
                               // USAP available?
                               YES: send args to USAP FD ──►
                                                         poll() returns
                                                         SpecializeCommon()
                                                           setresuid(10053)
                                                           setresgid(10053)
                                                           setgroups(...)
                                                           seccomp_filter
                                                           selinux_setcon
                                                         handleChildProc()
                                                           ZygoteInit.init()
                                                           → ActivityThread.main()
                               removeUsapTableEntry(i)
                               refillUsapPool()
                               fork() → USAP child #3
  ◄── pid=4567 ──────────────
  mProcessList.handleProcessStarted(pid)
  (watch for Binder connection from child)
                                                         Binder.joinThreadPool()
                                                         ◄── connect to AMS
```
