# Multi-Container Runtime

## 1. Team Information

| Name | SRN |
|------|-----|
| Sagar K | PES2UG24CS424 |
| S Vinyas Gangadkar | PES2UG24CS418 |

---

## 2. Build, Load, and Run Instructions

### Prerequisites

- Ubuntu 22.04 or 24.04 (bare metal or VM)
- Secure Boot **OFF** (required for kernel module loading)
- Linux kernel headers installed

```bash
sudo apt update
sudo apt install -y build-essential linux-headers-$(uname -r)
```

### Build

```bash
cd boilerplate

# Build all user-space binaries
make ci

# Build kernel module (requires kernel headers)
sudo make module

# Verify binaries exist
ls engine cpu_hog io_pulse memory_hog monitor.ko
```

### Set Up Root Filesystem

```bash
cd boilerplate

mkdir -p rootfs-base
wget https://dl-cdn.alpinelinux.org/alpine/v3.20/releases/x86_64/alpine-minirootfs-3.20.3-x86_64.tar.gz
sudo tar -xzf alpine-minirootfs-3.20.3-x86_64.tar.gz -C rootfs-base

# Create per-container writable copies
sudo cp -a rootfs-base rootfs-alpha
sudo cp -a rootfs-base rootfs-beta

# Copy workload binaries into rootfs copies
sudo cp cpu_hog memory_hog io_pulse rootfs-alpha/
sudo cp cpu_hog memory_hog io_pulse rootfs-beta/
```

### Load Kernel Module

```bash
sudo insmod monitor.ko

# Verify device was created
ls -la /dev/container_monitor

# Check kernel log
sudo dmesg | tail -3
```

### Start Supervisor

```bash
sudo rm -f /tmp/mini_runtime.sock
sudo ./engine supervisor ./rootfs-base
```

### Launch Containers

```bash
# In a second terminal:

# Start a container in background
sudo ./engine start alpha ./rootfs-alpha /cpu_hog

# Start a container and wait for it to finish
sudo ./engine run beta ./rootfs-beta /cpu_hog

# Start with custom memory limits and priority
sudo ./engine start memtest ./rootfs-alpha /memory_hog \
    --soft-mib 10 --hard-mib 20 --nice 5
```

### CLI Commands

```bash
# List all containers and their state
sudo ./engine ps

# View container logs
sudo ./engine logs alpha

# Stop a running container
sudo ./engine stop alpha
```

### Unload Module and Clean Up

```bash
# Stop supervisor first
sudo pkill -f "engine supervisor"
sudo rm -f /tmp/mini_runtime.sock

# Unload kernel module
sudo rmmod monitor

# Verify clean unload
sudo dmesg | tail -3
```

### Full Reference Run

```bash
# 1. Build
cd boilerplate
make ci
sudo make module

# 2. Load kernel module
sudo insmod monitor.ko
ls -l /dev/container_monitor

# 3. Start supervisor (Terminal 1)
sudo ./engine supervisor ./rootfs-base

# 4. Launch containers (Terminal 2)
sudo ./engine start alpha ./rootfs-alpha /cpu_hog
sudo ./engine start beta  ./rootfs-beta  /io_pulse

# 5. Inspect
sudo ./engine ps
sudo ./engine logs alpha
sudo ./engine logs beta

# 6. Test memory limits
sudo ./engine start memtest ./rootfs-alpha /memory_hog \
    --soft-mib 10 --hard-mib 20
sleep 8
sudo dmesg | grep container_monitor | tail -10

# 7. Stop containers
sudo ./engine stop alpha
sudo ./engine stop beta
sudo ./engine ps

# 8. Shutdown supervisor (Ctrl+C in Terminal 1)

# 9. Unload module
sudo rmmod monitor
sudo dmesg | tail -3
```

---
## 3. Demo Screenshot
All demo screenshots for Tasks 1–6 are included in this repository under the `boilerplate/screenshots/` directory.


## 4. Engineering Analysis

### 4.1 Isolation Mechanisms

Our runtime uses three Linux namespace types to isolate each container:

**PID namespace (`CLONE_NEWPID`):** Each container sees only its own processes. PID 1 inside the container is the container's init process. The host kernel maintains the real PID mapping — the container cannot see or signal host processes.

**UTS namespace (`CLONE_NEWUTS`):** Each container gets its own hostname. We call `sethostname(container_id)` in `child_fn()` so `hostname` inside the container returns the container ID rather than the host's hostname.

**Mount namespace (`CLONE_NEWNS`):** Each container gets its own mount table. We `chroot()` into the container's private rootfs copy and mount a fresh `/proc` with `MS_NOEXEC|MS_NOSUID|MS_NODEV` flags. This ensures tools like `ps` inside the container only see processes within the PID namespace.

We use `chroot()` rather than `pivot_root()` for simplicity. The host kernel still shares the underlying hardware, kernel code, and physical memory — namespaces partition the kernel's view of resources, not the kernel itself.

### 4.2 Supervisor and Process Lifecycle

The supervisor is a long-running parent process that never exits until explicitly stopped. This design is necessary because:

1. **Orphan prevention:** If the parent exits, container children become orphans adopted by init (PID 1), losing all metadata tracking.
2. **SIGCHLD delivery:** The kernel delivers SIGCHLD only to the direct parent. Our `sigchld_handler` calls `waitpid(-1, WNOHANG)` in a loop to reap all exited children atomically, preventing zombie accumulation.
3. **Metadata ownership:** The supervisor owns the `container_record_t` linked list. Each record tracks PID, state, limits, log path, and exit status. The list is protected by `metadata_lock` (a `pthread_mutex_t`) since both the main event loop and the SIGCHLD handler access it concurrently.

Container lifecycle transitions:
- `starting` → `running` (after `clone()` succeeds)
- `running` → `exited` (normal exit, detected via SIGCHLD)
- `running` → `stopped` (manual stop via `engine stop`, `stop_requested=1` set before SIGTERM)
- `running` → `killed` (hard-limit kill from kernel monitor, `stop_requested=0`)

### 4.3 IPC, Threads, and Synchronization

The project uses two distinct IPC mechanisms:

**Path A — Logging (container → supervisor):**
Each container's stdout and stderr are connected to the write end of a `pipe()`. A dedicated producer thread per container reads from the pipe and inserts `log_item_t` chunks into the bounded buffer. A single consumer thread (the logger thread) drains the buffer and writes to per-container log files.

The bounded buffer uses:
- `pthread_mutex_t mutex` — mutual exclusion on `head`, `tail`, `count`
- `pthread_cond_t not_empty` — consumer waits here when buffer is empty
- `pthread_cond_t not_full` — producer waits here when buffer is full

Without the mutex, two producer threads could simultaneously read `tail=5`, both write to slot 5, and one chunk would be silently overwritten. Without condition variables, threads would spin-poll wasting CPU. The `shutting_down` flag combined with `pthread_cond_broadcast()` ensures clean drain on shutdown — no log lines are lost.

**Path B — Control (CLI → supervisor):**
The CLI client connects to a UNIX domain socket at `/tmp/mini_runtime.sock`, sends a `control_request_t` struct, and reads back a `control_response_t`. For `CMD_RUN`, the supervisor `dup()`s the client fd and stores it in `run_client_fd` — the SIGCHLD handler writes the final exit status through this fd when the container exits.

### 4.4 Memory Management and Enforcement

**RSS (Resident Set Size)** measures the physical RAM pages currently mapped into a process's address space. It excludes swapped-out pages and shared library pages counted once per library — RSS represents actual memory pressure the process exerts on the system.

RSS does not measure virtual memory size, memory-mapped files that haven't been faulted in, or kernel memory allocated on behalf of the process.

**Soft vs hard limits serve different policy goals:**
- Soft limit triggers a warning logged to `dmesg`. The process continues running. This gives the operator visibility into memory growth before it becomes critical.
- Hard limit sends SIGKILL immediately. This is a hard safety boundary that prevents one container from exhausting host memory and triggering the OOM killer.

**Enforcement belongs in kernel space** because a user-space monitor can be outraced: if `memory_hog` allocates 100MB in one `malloc()` call, a 1-second polling loop in user space cannot react before the allocation succeeds. The kernel module's timer fires every second and checks RSS directly from the kernel's `mm_struct` — the same data structure the memory allocator updates. Additionally, only kernel code can reliably `SIGKILL` a process from a different namespace and guarantee delivery.

### 4.5 Scheduling Behavior

**Experiment 1 — nice=0 vs nice=15:**
Both containers ran `cpu_hog` for 10 seconds simultaneously. Wall-clock completion time was identical (~9.7s each) on our multi-core system because both containers received enough CPU time within the 10-second window. Under sustained CPU saturation, the Linux CFS scheduler assigns CPU share proportional to weight. nice=0 has weight 1024, nice=15 has weight 149 — a ratio of approximately 6.9:1, meaning the high-priority container would receive ~87% of CPU time and the low-priority container ~13% when competing on a single core.

**Experiment 2 — CPU-bound vs I/O-bound:**
`cpu_hog` (CPU-bound) ran for the full 10 seconds. `io_pulse` (I/O-bound) completed all 20 iterations in approximately 4 seconds. The I/O-bound workload finished 2.5x faster because it voluntarily yields the CPU during `usleep()` calls between iterations. CFS tracks virtual runtime — a sleeping process accumulates a "deficit" and gets boosted priority when it wakes up. This ensures I/O-bound tasks get CPU immediately on wake-up, maintaining responsiveness while CPU-bound tasks fill the remaining time.

---

## 5. Design Decisions and Tradeoffs

### Namespace isolation — chroot vs pivot_root
**Choice:** `chroot()` into per-container rootfs.
**Tradeoff:** `chroot` does not prevent a privileged process from escaping via `..` traversal. `pivot_root` is more secure but requires a separate bind-mount setup.
**Justification:** For this project's scope (trusted workloads, educational context), `chroot` is sufficient and significantly simpler to implement correctly. The mount namespace (`CLONE_NEWNS`) provides additional isolation of the mount table.

### Supervisor architecture — single process with threads
**Choice:** One supervisor process with a logging consumer thread and per-container producer threads.
**Tradeoff:** Threads share the address space, so a bug in one thread can corrupt shared state. A multi-process design would be more fault-isolated.
**Justification:** Shared memory simplifies the bounded buffer implementation — no IPC needed between producer and consumer. The mutex+condvar design correctly handles all race conditions.

### IPC mechanism — UNIX domain socket
**Choice:** UNIX domain socket at `/tmp/mini_runtime.sock` for CLI→supervisor control.
**Tradeoff:** The socket file persists on crash and must be manually cleaned up. A FIFO would be simpler but supports only one-directional communication.
**Justification:** UNIX sockets support bidirectional communication over a single connection, which is required for `CMD_RUN` (blocking until exit) and `CMD_LOGS` (streaming file contents back to the client).

### Kernel monitor — mutex vs spinlock
**Choice:** `DEFINE_MUTEX(list_lock)` to protect the monitored list.
**Tradeoff:** A spinlock would be faster for very short critical sections but cannot be held while sleeping. A mutex can sleep waiting for the lock.
**Justification:** The register path calls `kmalloc(GFP_KERNEL)` which can sleep — this is only safe with a mutex. The timer callback uses `mutex_trylock()` to avoid sleeping in softirq context. If the lock is held, the timer simply skips that tick and retries next second.

### Scheduling experiments — nice values
**Choice:** Used `nice()` syscall inside the container for priority control.
**Tradeoff:** `nice()` affects the entire container process. CPU affinity (`sched_setaffinity`) would give finer control but is harder to demo with short workloads.
**Justification:** nice values directly map to CFS weight, making the scheduler behavior predictable and explainable. The `--nice N` flag is exposed through the CLI making experiments reproducible.

---

## 6. Scheduler Experiment Results

### Experiment 1 — Two CPU-bound containers with different priorities

| Container | nice value | CFS weight | Wall-clock time | Notes |
|-----------|-----------|------------|-----------------|-------|
| alpha | 0 | 1024 | 9.725s | Default priority |
| beta | 15 | 149 | 9.726s | Low priority |

Both completed in the same wall-clock time on our multi-core system because the workload duration (10s) was short enough for both to finish without starvation. Under sustained load on a single core, CFS would allocate CPU in ratio 1024:149 ≈ 6.9:1, giving alpha ~87% and beta ~13% of CPU time.

**Conclusion:** The Linux CFS scheduler uses weighted fair queuing. Under light load, all runnable processes get CPU. Under heavy contention, nice values directly control CPU share allocation.

### Experiment 2 — CPU-bound vs I/O-bound containers

| Container | Workload | Duration | Behaviour |
|-----------|----------|----------|-----------|
| cpuwork | cpu_hog | 10s | Never sleeps, always runnable |
| iowork | io_pulse | ~4s | Sleeps 200ms between iterations |

`io_pulse` completed all 20 iterations in approximately 4 seconds despite running alongside a CPU-bound workload. The I/O-bound container spent most of its time sleeping (200ms × 20 = 4s total sleep), voluntarily yielding the CPU to `cpu_hog` between iterations.

When `io_pulse` woke from sleep, CFS immediately scheduled it because its virtual runtime was lower than `cpu_hog`'s (it had been sleeping, accumulating a scheduling credit). This demonstrates CFS's responsiveness guarantee — I/O-bound tasks are never starved by CPU-bound tasks.

**Conclusion:** The Linux CFS scheduler naturally favours I/O-bound tasks by giving them scheduling credit during sleep periods. CPU-bound tasks fill available CPU time without blocking I/O-bound tasks from getting timely service.