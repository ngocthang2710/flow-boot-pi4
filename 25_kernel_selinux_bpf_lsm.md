# Kernel SELinux & BPF LSM — Raspberry Pi 4

## 1. Tổng quan

Android dùng SELinux để enforce mandatory access control. Trên Pi4 còn có thêm BPF LSM (eBPF-based security) và Landlock (sandboxing). Cả 3 đều đang enabled.

**Config đã enabled:**
```
CONFIG_SECURITY=y
CONFIG_SECURITY_SELINUX=y           ← SELinux (Android mandatory)
CONFIG_SECURITY_SELINUX_DEVELOP=y   ← Permissive mode support
CONFIG_BPF_LSM=y                    ← eBPF-based LSM hooks
CONFIG_SECURITY_LANDLOCK=y          ← Landlock sandboxing
CONFIG_INTEGRITY=y                  ← Integrity subsystem
CONFIG_IMA=y                        ← Integrity Measurement Architecture
CONFIG_AUDIT=y                      ← Audit log
CONFIG_AUDITSYSCALL=y               ← Syscall auditing
CONFIG_SECURITY_LOADPIN=y           ← Restrict kernel module loading
```

**Files:**
```
kernel/common/security/selinux/     ← SELinux implementation (50+ files)
kernel/common/kernel/bpf/bpf_lsm.c ← BPF LSM hooks
kernel/common/security/landlock/    ← Landlock (6 files)
kernel/common/security/integrity/ima/ ← IMA
```

---

## 2. SELinux trên Pi4 Android

### 2.1 Trạng thái hiện tại

Pi4 build chạy SELinux ở **permissive mode** (theo kernel cmdline):
```
androidboot.selinux=permissive
```

Có nghĩa: SELinux log denials nhưng **không enforce** — phù hợp để develop.

```bash
# Kiểm tra SELinux mode
getenforce
# Permissive

# Xem SELinux status
sestatus

# Chuyển sang enforcing (cần policy đúng)
setenforce 1
getenforce
# Enforcing

# Chuyển lại permissive
setenforce 0
```

---

### 2.2 SELinux Policy trên Android

```bash
# Policy files được compile vào system partition
ls /system/etc/selinux/
# plat_sepolicy.cil
# plat_file_contexts
# plat_property_contexts
# plat_service_contexts

# Policy cho Pi4-specific (vendor)
ls /vendor/etc/selinux/
# vendor_sepolicy.cil
# vendor_file_contexts

# Load policy
ls /sys/fs/selinux/
# enforce
# policy
# status
```

---

### 2.3 Xem SELinux Denials

```bash
# Xem audit log (denials)
dmesg | grep "avc: denied"
# avc: denied { read } for pid=1234
#   comm="my_daemon" name="gpio17" dev="sysfs"
#   scontext=u:r:my_daemon:s0
#   tcontext=u:object_r:sysfs_gpio:s0
#   tclass=file permissive=1

# Audit log
cat /proc/kmsg | grep "avc:"

# Dùng audit2allow để generate policy
dmesg | grep "avc:" | audit2allow -M my_policy
```

---

### 2.4 Viết SELinux Policy cho Pi4 driver

```te
# my_gpio_policy.te

# Định nghĩa type cho daemon của mình
type my_gpio_daemon, domain;
type my_gpio_daemon_exec, exec_type, vendor_file_type, file_type;

# Init transition
init_daemon_domain(my_gpio_daemon)

# Cho phép đọc GPIO sysfs
allow my_gpio_daemon sysfs_gpio:file { read write open };
allow my_gpio_daemon sysfs_gpio:dir { read search };

# Cho phép mở /dev/gpiochip0
allow my_gpio_daemon gpio_device:chr_file { read write open ioctl };

# Cho phép I2C access
allow my_gpio_daemon i2c_device:chr_file { read write open ioctl };

# Logging
allow my_gpio_daemon kmsg_device:chr_file { open write };
```

```makefile
# device/brcm/rpi4/sepolicy/Android.mk
BOARD_SEPOLICY_DIRS += device/brcm/rpi4/sepolicy
```

---

### 2.5 File Contexts

```
# device/brcm/rpi4/sepolicy/file_contexts

/dev/gpiochip[0-9]+   u:object_r:gpio_device:s0
/dev/i2c-[0-9]+       u:object_r:i2c_device:s0
/dev/spidev[0-9]+\.[0-9]+  u:object_r:spi_device:s0
/sys/class/gpio(/.*)?  u:object_r:sysfs_gpio:s0
/sys/class/pwm(/.*)?   u:object_r:sysfs_pwm:s0
/sys/class/thermal(/.*)?  u:object_r:sysfs_thermal:s0
```

---

## 3. BPF LSM — Programmable Security

BPF LSM cho phép implement security policy bằng eBPF program, attach vào LSM hooks.

### 3.1 Các LSM hooks có thể attach

```c
/* Từ kernel/bpf/bpf_lsm.c */
/* Một số hooks quan trọng: */

LSM_HOOK(int, 0, file_open, struct file *file)
LSM_HOOK(int, 0, file_permission, struct file *file, int mask)
LSM_HOOK(int, 0, inode_create, struct inode *dir, struct dentry *dentry, umode_t mode)
LSM_HOOK(int, 0, socket_connect, struct socket *sock, struct sockaddr *address, int addrlen)
LSM_HOOK(int, 0, bpf, int cmd, union bpf_attr *attr, unsigned int size)
LSM_HOOK(int, 0, task_kill, struct task_struct *p, struct kernel_siginfo *info, int sig)
LSM_HOOK(int, 0, sb_mount, const char *dev_name, const struct path *path, const char *type)
```

### 3.2 Ví dụ BPF LSM Program

```c
/* restrict_gpio_access.bpf.c */
#include <linux/bpf.h>
#include <bpf/bpf_helpers.h>
#include <bpf/bpf_tracing.h>

/* Whitelist UIDs được phép dùng GPIO */
struct {
    __uint(type, BPF_MAP_TYPE_HASH);
    __type(key,  u32);   /* UID */
    __type(value, u8);   /* allowed */
    __uint(max_entries, 16);
} allowed_uids SEC(".maps");

SEC("lsm/file_open")
int BPF_PROG(restrict_gpio_open, struct file *file)
{
    char fname[64];
    int ret;

    /* Lấy filename */
    ret = bpf_d_path(&file->f_path, fname, sizeof(fname));
    if (ret < 0)
        return 0;

    /* Chỉ enforce với /dev/gpiochip* */
    if (bpf_strncmp(fname, 13, "/dev/gpiochip") != 0)
        return 0;

    /* Kiểm tra UID */
    u32 uid = bpf_get_current_uid_gid() & 0xFFFFFFFF;
    u8 *allowed = bpf_map_lookup_elem(&allowed_uids, &uid);

    if (!allowed || *allowed == 0) {
        bpf_printk("BPF LSM: denied GPIO access for UID %d\n", uid);
        return -EPERM;
    }

    return 0;
}

SEC("lsm/socket_connect")
int BPF_PROG(audit_connections, struct socket *sock,
             struct sockaddr *address, int addrlen)
{
    /* Log tất cả TCP connections */
    u32 pid = bpf_get_current_pid_tgid() >> 32;
    char comm[16];
    bpf_get_current_comm(comm, sizeof(comm));

    bpf_printk("TCP connect: pid=%d comm=%s\n", pid, comm);
    return 0;
}

char LICENSE[] SEC("license") = "GPL";
```

---

## 4. Landlock — Process Sandboxing

Landlock cho phép một process tự giới hạn quyền truy cập file system của nó — không cần root.

```c
/* landlock_sandbox.c — Sandbox chỉ cho phép đọc /sys/class/gpio */
#include <linux/landlock.h>
#include <sys/syscall.h>
#include <fcntl.h>
#include <unistd.h>

#define LANDLOCK_CREATE _IO(LANDLOCK_IOC_MAGIC, 0)
#define sys_landlock_create_ruleset  445
#define sys_landlock_add_rule        446
#define sys_landlock_restrict_self   447

int main(void)
{
    /* Tạo ruleset: chỉ cho phép read file */
    struct landlock_ruleset_attr rs_attr = {
        .handled_access_fs = LANDLOCK_ACCESS_FS_READ_FILE |
                             LANDLOCK_ACCESS_FS_READ_DIR,
    };

    int rs_fd = syscall(sys_landlock_create_ruleset,
                        &rs_attr, sizeof(rs_attr), 0);

    /* Thêm rule: được phép đọc /sys/class/gpio */
    int gpio_fd = open("/sys/class/gpio", O_PATH | O_CLOEXEC);
    struct landlock_path_beneath_attr rule = {
        .allowed_access = LANDLOCK_ACCESS_FS_READ_FILE |
                          LANDLOCK_ACCESS_FS_READ_DIR,
        .parent_fd = gpio_fd,
    };
    syscall(sys_landlock_add_rule, rs_fd,
            LANDLOCK_RULE_PATH_BENEATH, &rule, 0);

    /* Enforce sandbox */
    prctl(PR_SET_NO_NEW_PRIVS, 1, 0, 0, 0);
    syscall(sys_landlock_restrict_self, rs_fd, 0);

    close(rs_fd);
    close(gpio_fd);

    /* Từ đây process chỉ có thể đọc /sys/class/gpio */
    /* Mở /etc/passwd sẽ bị EPERM */

    char buf[64];
    int fd = open("/sys/class/gpio/gpio17/value", O_RDONLY);
    read(fd, buf, sizeof(buf));
    printf("GPIO value: %s\n", buf);

    /* Bị từ chối */
    fd = open("/etc/passwd", O_RDONLY);
    if (fd < 0)
        printf("Access to /etc/passwd denied (expected)\n");

    return 0;
}
```

---

## 5. IMA — Integrity Measurement Architecture

```bash
# IMA đo và log hash của files được execute/read
# Xem IMA policy
cat /sys/kernel/security/ima/policy

# Xem measurement log
cat /sys/kernel/security/ima/ascii_runtime_measurements
# PCR  template-hash filedata-hash filename
# 10  sha256:abc... sha256:def... /system/bin/surfaceflinger
# 10  sha256:...    sha256:...    /vendor/bin/hw/android.hardware.audio@...

# Xem IMA violations
cat /sys/kernel/security/ima/violations
```

---

## 6. Audit Subsystem

```bash
# Xem audit rules
auditctl -l

# Audit tất cả write vào GPIO sysfs
auditctl -w /sys/class/gpio -p w -k gpio_writes

# Xem audit log
ausearch -k gpio_writes
# time->...
# type=SYSCALL msg=audit(1234): arch=aarch64 syscall=write
#   success=yes pid=1234 comm="my_app" exe="/system/bin/my_app"
# type=PATH item=0 name="/sys/class/gpio/gpio17/value"

# Audit process execution
auditctl -a always,exit -F arch=aarch64 -S execve -k process_exec
```

---

## 7. Thực hành: Security Policy cho Pi4

```bash
# 1. Chạy trong permissive mode, quan sát denials
setenforce 0
# Chạy app/daemon...
dmesg | grep "avc: denied" > denials.txt

# 2. Convert denials thành policy
cat denials.txt | audit2allow

# 3. Harden: enforce chỉ đúng permissions cần thiết
setenforce 1
# Test từng tính năng

# 4. Xem context của process
ps -Z | grep my_daemon
# u:r:my_daemon:s0   1234  ...  my_daemon

# 5. Xem context của file
ls -Z /sys/class/gpio/
# u:object_r:sysfs_gpio:s0  gpio17/

# 6. Kiểm tra access thủ công
sesearch --allow -s my_daemon -c file
```

---

## 8. SELinux Policy trong source Pi4

```
source/device/brcm/rpi4/sepolicy/
├── file_contexts       ← Gán context cho files
├── property_contexts   ← Context cho system properties
├── service_contexts    ← Context cho Android services
├── my_daemon.te        ← Policy cho custom daemon
└── Android.mk          ← Include vào build
```

---

## 9. So sánh 3 security mechanisms

| Cơ chế | Approach | Granularity | Dùng cho |
|--------|----------|------------|---------|
| SELinux | Policy labels | Process + file + socket | Android mandatory |
| BPF LSM | eBPF program | Bất kỳ LSM hook | Custom dynamic policy |
| Landlock | Self-restriction | File path | Sandbox từng process |

---

## 10. File quan trọng trong source

| File | Mô tả |
|------|-------|
| `security/selinux/hooks.c` | SELinux LSM hooks (12000+ lines) |
| `security/selinux/avc.c` | Access Vector Cache |
| `security/selinux/ss/` | Security server, policy engine |
| `kernel/bpf/bpf_lsm.c` | BPF LSM hook registration |
| `security/landlock/fs.c` | Landlock filesystem rules |
| `security/landlock/ruleset.c` | Landlock ruleset management |
| `security/integrity/ima/ima_main.c` | IMA measurement |
| `kernel/audit.c` | Audit subsystem core |
| `device/brcm/rpi4/sepolicy/` | Pi4 SELinux policy |
