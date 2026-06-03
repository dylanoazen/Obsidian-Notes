# Linux Internals

A deep dive into how Linux works under the hood — from power button to running process.

This is not about distros. This is about the operating system itself: how it boots, how the kernel manages hardware, memory, processes, and everything in between.

---

## The Boot Process

What happens from power on to login prompt:

```
Power On
  │
  ▼
BIOS/UEFI ──► POST (Power-On Self-Test)
  │              checks CPU, RAM, devices
  ▼
Bootloader (GRUB2)
  │   loads kernel + initramfs into memory
  ▼
Kernel Initialization
  │   decompresses, sets up memory, detects hardware
  ▼
initramfs (Initial RAM Filesystem)
  │   temporary root filesystem in memory
  │   loads essential drivers to mount real root
  ▼
Init System (systemd / PID 1)
  │   first userspace process
  │   starts all services and targets
  ▼
Login Prompt / Desktop
```

### BIOS vs UEFI

- **BIOS** (legacy): 16-bit, MBR partition table, max 2TB disks, sequential boot
- **UEFI**: 64-bit, GPT partition table, no size limit, Secure Boot, faster
- UEFI stores bootloaders in the **ESP** (EFI System Partition), usually at `/boot/efi`

### GRUB2

- Grand Unified Bootloader — most common Linux bootloader
- Config at `/boot/grub/grub.cfg` (generated, don't edit directly)
- Edit `/etc/default/grub` → run `grub-mkconfig -o /boot/grub/grub.cfg`
- Can chainload other OS bootloaders (dual boot)

### initramfs

- Compressed archive loaded into RAM alongside the kernel
- Contains minimal filesystem with drivers needed to mount the real root
- Why? Kernel image can't contain every possible disk/filesystem driver
- After real root is mounted, initramfs is discarded

## Topics

- [[Linux/Kernel]]
- [[Linux/ProcessManagement]]
- [[Linux/MemoryManagement]]
- [[Linux/FileSystem]]
- [[Linux/SystemCalls]]
- [[Linux/Networking]]
- [[Linux/ProcessesAndThreads]]

## Resources

- https://0xax.gitbooks.io/linux-insides/content/ (Linux Insides — deep dive)
- https://tldp.org/LDP/tlk/tlk.html (The Linux Kernel)
- https://man7.org/linux/man-pages/ (man pages online)
- Linux Kernel Development — Robert Love (book)
- Operating Systems: Three Easy Pieces — free online textbook

#### My commentaries
- 
