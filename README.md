## Multi-Container Runtime

A lightweight Linux container runtime in C with a long-running parent supervisor and a kernel-space memory monitor (LKM). Implements process isolation via Linux namespaces, concurrent bounded-buffer logging, a CLI control plane over UNIX domain sockets, and kernel-enforced memory limits.

Team Information:

NAME: ANUPRIYA
SRN: PES2UG24CS074


NAME: ANVII S RAI
SRN: PES2UG24CS076

## Getting Started

### Build & Setup
Prerequisites

The project requires an Ubuntu 22.04 or 24.04 VM with Secure Boot OFF. 
Note that WSL is not supported due to the custom kernel module requirements.
```
sudo apt update
sudo apt install -y build-essential linux-headers-$(uname -r)
```
Compilation
Run the following in the boilerplate directory:
```
make
```
This compiles the Engine (supervisor), the Monitor (kernel module), and the test workloads (cpu_hog, io_pulse, and memory_hog).

Note: For a CI-safe user-space-only check, use make -C boilerplate ci.

Prepare Root Filesystems
```
mkdir rootfs-base
wget https://dl-cdn.alpinelinux.org/alpine/v3.20/releases/x86_64/alpine-minirootfs-3.20.3-x86_64.tar.gz
tar -xzf alpine-minirootfs-3.20.3-x86_64.tar.gz -C rootfs-base
# Create writable copies
cp -a ./rootfs-base ./rootfs-alpha
cp -a ./rootfs-base ./rootfs-beta
```

## Execution Phases (Demo Workflow)

***Phase 1: Setup & Supervisor***
Load Module:
```
sudo insmod monitor.ko
lsmod | grep monitor        # Verify loading
ls -l /dev/container_monitor # Verify device node
```

Start Supervisor (Terminal 1):
```
sudo ./engine supervisor ../rootfs-base
```
***Phase 2: Isolation & Scheduling (Tasks 1, 2 & 5)***
Launch Containers (Terminal 2):

```
sudo ./engine start alpha ../rootfs-alpha "sleep 200"
sudo ./engine start beta ../rootfs-beta "sleep 200"
```
Verify State:
```
sudo ./engine ps
```

***Phase 3: Logging & Monitoring (Tasks 3 & 4)***

Test Logging Persistence:
```
sudo ./engine start logger1 ../rootfs-alpha "sh -c 'for i in {1..10}; do echo 1234; sleep 1; done'"
sudo ./engine logs logger1
```
Verify Kernel Monitoring:
```
sudo dmesg | grep -i "monitor"
```
***Phase 4: Cleanup (Task 6)***

Stop Processes:
```
sudo ./engine stop alpha
sudo ./engine stop beta
sudo ./engine stop logger1
sudo ./engine ps
```

Shutdown Supervisor: Switch to Terminal 1 and press Ctrl+C.

Final Unload:
```
sudo rmmod monitor.
sudo dmesg | grep "container_monitor" | tail -n 5
```
### 5. Understand the Boilerplate

The `boilerplate/` folder contains starter files:

| File                   | Purpose                                             |
| ---------------------- | --------------------------------------------------- |
| `engine.c`             | User-space runtime and supervisor skeleton          |
| `monitor.c`            | Kernel module skeleton                              |
| `monitor_ioctl.h`      | Shared ioctl command definitions                    |
| `Makefile`             | Build targets for both user-space and kernel module |
| `cpu_hog.c`            | CPU-bound test workload                             |
| `io_pulse.c`           | I/O-bound test workload                             |
| `memory_hog.c`         | Memory-consuming test workload                      |
| `environment-check.sh` | VM environment preflight check                      |


### 6. Build and Verify

```bash
cd boilerplate
make
```

The CI-safe build command is:

```bash
make -C boilerplate ci
```
---
## DEMO & SCREENSHOTS 

***Task 1 – Starting Two Containers Commands run:***

sudo ./engine start alpha ../rootfs-alpha "sleep 200" sudo ./engine start beta ../rootfs-beta "sleep 200"
Both containers started in their own namespaces. The supervisor in Terminal 1 printed lifecycle and /proc mount messages for each.


 
Terminal 1
 
Terminal 2
 
Task 2 – PS Command Output Command run:
sudo ./engine ps
Shows alpha and beta tracked with their host PIDs, RUNNING state, start times, log paths, and 40/64 MiB memory limits.
 

Task 3 – Logging Pipeline
Commands run:
sudo ./engine start logger1 ../rootfs-alpha "sh -c 'for i in {1..10}; do echo 1234; sleep 1; done'"
# Wait ~10 seconds sudo ./engine logs logger1
Log file shows ten lines of '1234' captured from the container's stdout via the bounded-buffer pipeline.

 

Task 3 (IPC) – CLI and UNIX Socket
Terminal 1 output shows the IPC messages — [IPC] Received CMD_START, CMD_LOGS, CMD_PS — confirming commands arrive over the UNIX socket and the supervisor responds to each one.

 

Task 4 – Soft Limit Warning and Hard Limit Kill Commands run:
sudo ./engine start hogger ../rootfs-alpha "./memory_hog" sudo dmesg | grep -i "monitor"
dmesg shows the SOFT LIMIT line from the kernel module when hogger's RSS crossed 40 MiB.
The same dmesg output shows the HARD LIMIT line when RSS crossed 64 MiB, followed by the kill. The supervisor's ps then reflects hogger's state as KILLED (HARD LIMIT).
 


Task 5 – Scheduling Experiment Command run:
sudo ./engine ps
PS output showing containers running concurrently. Timing data from cpu_hog runs demonstrates CFS fair sharing at equal nice values and faster completion for the lower-nice container.
 


Task 6 – Clean Teardown Commands run:
sudo ./engine stop alpha sudo ./engine stop beta sudo ./engine ps
# Ctrl+C in Terminal 1 sudo rmmod monitor sudo dmesg | tail -n 1
PS shows STOPPED state. Terminal 1 shows clean supervisor exit. dmesg confirms module unloaded. No zombies in ps aux.

 





---
## Key Features & Tasks

***1. Multi-Container Isolation (Tasks 1 & 2)***

The runtime achieves true process-level isolation by leveraging the clone() system call with CLONE_NEWPID, CLONE_NEWUTS, and CLONE_NEWNS flags. This setup ensures every container operates in its own sandbox with a private hostname and process tree.

Filesystem Jail: Using chroot(), containers are locked into an Alpine minirootfs, preventing any "directory escape" to the host system.

Proc Separation: By mounting a private /proc for each container, we ensured that processes inside alpha cannot see what's running in beta or on the host.

The Result: During testing, we successfully ran concurrent containers with distinct host PIDs, proving the namespace separation was airtight.

***2. Asynchronous Bounded-Buffer Logging (Task 3)***

One of the biggest challenges was ensuring that logging didn't slow down the container. We solved this by building an asynchronous pipeline:

The Logic: Container output is funneled through unidirectional pipes into a bounded ring buffer.

Concurrency: A dedicated POSIX background thread handles the "heavy lifting" of writing to disk. We managed thread safety using a mutex and a condition variable to ensure the consumer thread doesn't waste CPU cycles when the buffer is empty.

The Proof: We verified this with high-frequency echo loops, confirming zero data loss and perfect message ordering.

***3. Kernel-Space Memory Monitoring (Task 4)***

For resource enforcement, we went deep into the kernel. We built a custom Linux Kernel Module (LKM) that tracks Resident Set Size (RSS) by accessing the process mm_struct directly.

Soft Limit (40 MiB): Acts as a "warning shot," triggering a message in dmesg.

Hard Limit (64 MiB): Acts as a kill-switch. The kernel module issues an unblockable SIGKILL to immediately stop a runaway process.

The Result: Our memory_hog tests proved that the monitor is extremely precise, terminating the process the second it touched the 64 MiB ceiling.

***4. CLI Control Plane (Task 5)***

Management is handled by a long-running Supervisor process. Instead of simple files, we used UNIX Domain Sockets to create a bidirectional communication channel between the CLI and the Engine.

This allowed us to implement reliable, structured commands like start, ps, logs, and stop.

In our tests, the ps command provided a consistent, real-time snapshot of every container’s state, host PID, and memory usage.

***5. Clean Teardown & Resource Recovery (Task 6)***

A major focus was ensuring the system didn't leave "ghosts" behind after closing.

Zombie Reaping: We programmed the supervisor to catch SIGCHLD signals and use waitpid() to properly "reap" exited containers, keeping the system process table clean.

Module Lifecycle: We designed a specific teardown sequence where the supervisor releases all handles to /dev/container_monitor before the module is unloaded.

The Proof: Post-test checks confirmed zero zombie processes and an error-free rmmod, with dmesg confirming a clean module unregistration.

---

## Engineering Analysis

***1. The Isolation Handshake***

Container isolation is a functional handshake between the Process Tree and the Filesystem. While the container operates as "PID 1" due to CLONE_NEWPID (Namespace isolation), it remains a standard child process on the host. This dual-view allows the Monitor to track it efficiently without the overhead of full virtualization.

***2. Kernel-Space Enforcement***

By accessing the mm_struct directly in the kernel, the Monitor detects memory spikes at the hardware level in real-time. This architecture enables the issuance of a SIGKILL that cannot be blocked, caught, or ignored by a malfunctioning process, providing a significantly higher security tier than standard user-space monitoring.

***3. Asynchronous Pipe Management***

The implementation of a Bounded Ring Buffer effectively decouples container execution from disk I/O. This design ensures that even during periods of high disk latency on the host machine, the container’s internal execution speed remains constant and unaffected by the logging pipeline.


***4. Synchronization & Race Condition Prevention***

The logging pipeline relies on a classic Producer-Consumer model. To prevent race conditions where the Engine (Producer) and the Logger Thread (Consumer) access the Ring Buffer simultaneously, the implementation utilizes POSIX Mutexes. Furthermore, a Condition Variable was implemented to solve the "busy-wait" problem, ensuring the consumer thread remains asleep until new data is pushed, saving significant CPU cycles.

***5. CPU Scheduling & Nice Values***

By conducting experiments with the Completely Fair Scheduler (CFS), the project analyzed how Linux distributes CPU time based on priority weights. The findings confirmed that lowering a container's nice value (increasing its priority) directly correlates to a larger share of CPU cycles. This integration proves that while our runtime provides isolation, it still respects the underlying kernel’s scheduling policies for resource fairness.
