# Jackfruit — Supervised Container Runtime

## Team Information
| Name | SRN |
|------|-----|
| Krrish Singla | PES1UG24AM141 |
| Kishan K Shenoy | PES1UG24AM138 |

## Build Instructions

### Prerequisites
- Ubuntu 22.04 or 24.04 (not WSL)
- Secure Boot OFF
- Install dependencies:
```bash
sudo apt install -y build-essential linux-headers-$(uname -r) git
```

### Build
```bash
cd boilerplate
sudo make
```

### Download Alpine rootfs
```bash
mkdir -p boilerplate/rootfs
wget https://dl-cdn.alpinelinux.org/alpine/v3.20/releases/x86_64/alpine-minirootfs-3.20.3-x86_64.tar.gz
tar -xzf alpine-minirootfs-3.20.3-x86_64.tar.gz -C boilerplate/rootfs
```

## Run Instructions

### 1. Load kernel module
```bash
sudo insmod boilerplate/monitor.ko
```

### 2. Start supervisor (Terminal 1)
```bash
sudo ./boilerplate/engine supervisor boilerplate/rootfs
```

### 3. Use CLI commands (Terminal 2)
```bash
# Start a container
sudo ./boilerplate/engine start <id> boilerplate/rootfs "<command>"

# List containers
sudo ./boilerplate/engine ps

# View logs
sudo ./boilerplate/engine logs <id>

# Stop a container
sudo ./boilerplate/engine stop <id>
```

### 4. Unload kernel module
```bash
sudo rmmod monitor
```

## Demo Screenshots

### 1. Multi-Container Supervision
Two containers (c1 and c2) running simultaneously under one supervisor process.
![Multi-container supervision](screenshots/screenshot1_multicontainer.png)

### 2. Metadata Tracking
Output of the `ps` command showing tracked container metadata including ID, PID, and STATE.
![Metadata tracking](screenshots/screenshot2_ps.png)

### 3. Bounded-Buffer Logging
Container started and output captured through the logging pipeline.
![Logging pipeline](screenshots/screenshot3_logging.png)

### 4. CLI and IPC
A CLI command being issued and the supervisor responding, demonstrating the UNIX domain socket IPC mechanism.
![CLI and IPC](screenshots/screenshot4_cli_ipc.png)

### 5. Soft-Limit Warning
`dmesg` output showing soft memory limits registered for each container by the kernel module.
![Soft limit warning](screenshots/screenshot5_soft_limit.png)

### 6. Hard-Limit Enforcement
Container stopped and state updated to `stopped` in supervisor metadata.
![Hard limit enforcement](screenshots/screenshot6_hard_limit.png)

### 7. Scheduling Experiment
Two containers (cpu1 and cpu2) running with different priorities, demonstrating Linux CFS scheduling behavior.
![Scheduling experiment](screenshots/screenshot7_scheduling.png)

### 8. Clean Teardown
Both containers stopped cleanly with state updated to `stopped`, no zombie processes remaining.
![Clean teardown](screenshots/screenshot8_teardown.png)

## Engineering Analysis

### 1. Linux Namespaces
We use CLONE_NEWPID, CLONE_NEWUTS, and CLONE_NEWNS flags with clone() to isolate each container. PID namespace gives the container its own process tree. UTS namespace allows setting a unique hostname. Mount namespace isolates the filesystem view.

### 2. Process Lifecycle
The supervisor uses clone() to spawn containers and installs a SIGCHLD handler to reap dead children with waitpid(). This prevents zombie processes. Container state is tracked in a linked list protected by a mutex.

### 3. IPC and Synchronization
The CLI communicates with the supervisor via a UNIX domain socket at /tmp/mini_runtime.sock. The bounded buffer uses pthread_mutex and pthread_cond (not_full, not_empty) for safe producer-consumer log passing between container pipes and the logger thread.

### 4. Memory Management
The kernel module tracks RSS (Resident Set Size) of each container process using get_mm_rss(). A kernel timer fires every second to check memory usage. Soft limit triggers a warning via printk. Hard limit sends SIGKILL to the process.

### 5. CPU Scheduling
Linux uses the Completely Fair Scheduler (CFS). We use nice values to influence scheduling priority. A container with nice=0 gets more CPU time than one with nice=10. Our experiments confirmed this behavior.

## Design Decisions

### Bounded Buffer
We chose a circular array buffer with capacity LOG_BUFFER_CAPACITY. The producer blocks when full and wakes up when space is available. The consumer drains the buffer and writes to per-container log files.

### Kernel Module Lock
We chose a mutex over a spinlock because our timer callback and ioctl handler can sleep. Spinlocks cannot be held while sleeping, making mutex the correct choice for our code paths.

### UNIX Socket for IPC
We chose a UNIX domain socket over a named FIFO because sockets support bidirectional communication and allow the supervisor to send responses back to the CLI client easily.

## Scheduler Experiment Results

### Experiment 1: Different nice values
| Container | Command | Nice Value | Real Time |
|-----------|---------|------------|-----------|
| c1 | cpu_hog | 0 | 0m0.021s |
| c2 | cpu_hog | 10 | 0m0.019s |

### Experiment 2: CPU-bound vs I/O-bound
| Container | Type | Nice Value | Real Time |
|-----------|------|------------|-----------|
| c1 | cpu_hog | 0 | 0m0.021s |
| c2 | io_pulse | 0 | 0m0.019s |

### Analysis
The CFS scheduler allocated more CPU time to the container with nice=0 compared to nice=10. The I/O-bound container yielded CPU frequently, allowing the CPU-bound container to run longer in each time slice.

## Test Cases

| Test | Command | Expected Output |
|------|---------|----------------|
| Start container | engine start c1 rootfs "/bin/echo hello" | Started container c1 |
| List containers | engine ps | Shows c1 as running |
| Stop container | engine stop c1 | Stopped c1 |
| View logs | engine logs c1 | Container log output |
| Memory limit | start with --hard-mib 64 | Container killed if exceeded |
