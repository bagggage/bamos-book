# Current Development Progress

1. [Main subsystems and components](#main-subsystems-and-components)
2. [Hardware drivers](#hardware-drivers)
3. [Software drivers](#software-drivers)
4. [ABI](#progress-on-abi)
- [GNU/Linux](#gnulinux)

## Main subsystems and components

| Feature | Status | Docs |
|---|---|---|
| x86-64 Support | **Most recent** ||
| Memory management | **Most recent** | [*Outdated*](./kernel/memory.md) |
| SMP | **Almost done** ||
| Device management | **Almost done** | [*Need updates*](./kernel/devices.md) |
| I/O subsystem | **Done** | [*Most recent*](./kernel/devices/io.md) |
| Interrupt handling | **Almost done** | [*Need updates*](./kernel/devices/interrupts.md) |
| Device classes subsystem | **Almost done** | [*Need updates*](./kernel/devices/classes.md) |
| Virtual file system | *In progress ||
| Processes and threads | *In progress ||
| System calls | *In progress ||
| Logging subsystem | **Most recent** | [*Most recent*](./kernel/logging.md) |

## Hardware drivers

| Driver | Status | Docs |
|---|---|---|
| PCI/PCI-E | **Almost done** ||
| NVMe | **Almost done** ||
| UART (8250) | **Done** ||
| AHCI | *Planned* ||
| USB (XHCI) | *Planned* ||
| PS/2 Keyboard | **Done** ||

## Software drivers

| Driver | Status | Docs |
|---|---|---|
| ext2 | *In progress ||
| tmpfs | *In progress ||
| initramfs | *In progress ||
| ext4 | *Planned* ||
| devfs | *In progress ||
| NTFS | *Planned* ||
| FAT32 | *Planned* ||
| /dev/tty | *In progress ||
| /dev/console | *In progress ||

# Progress on ABI

## GNU/Linux

| Syscall | Details |
|---|---|
| `arch_prctl` | not complete |
| `bkr` ||
| `clock_gettime` ||
| `fstat` ||
| `fstatat64` ||
| `get_robust_list` ||
| `getcwd` ||
| `getegid` | not complete |
| `geteuid` | not complete |
| `getgid` ||
| `getpid` ||
| `getppid` ||
| `getrandom` | not really complete, currently uses pseudo-random algorithm to generate numbers |
| `gettid` | thread id is not implemented yet |
| `getuid` ||
| `ioctl` ||
| `mmap` ||
| `mprotect` ||
| `open` ||
| `pread64` ||
| `preadv` ||
| `prlimit64` ||
| `pwrite64` ||
| `pwritev` ||
| `read` ||
| `readv` ||
| `rseq` | only set a pointer, the sequence itself not handled yet |
| `set_robust_list` | only set an array pointer, `futex` is not implemented |
| `set_tid_address` ||
| `stat` ||
| `time` ||
| `uname` ||
| `write` ||
| `writev` ||
