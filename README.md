# 32-bit Protected-Mode Operating System — Advanced + AI Edition

A **32-bit protected-mode operating system** developed in C/C++ with advanced OS features including **Ring 3 user mode, process management, IPC, authentication, file permissions, multiple CPU scheduling algorithms, and AI-assisted CPU scheduling**.

The project demonstrates how traditional operating-system concepts can be extended with **machine learning to make CPU scheduling adaptive**.

---

## 🚀 Features

### 🔐 Protected Mode & Privilege Levels

* 32-bit protected-mode architecture
* GDT and IDT
* Ring 0 Kernel Mode
* Ring 3 User Mode
* TSS-based Ring 3 → Ring 0 transitions
* User/kernel memory isolation

### ⚙️ Process Management

* Process creation and termination
* `fork()` system call
* Process monitoring
* Context switching
* Preemptive scheduling support

### 🔄 Inter-Process Communication (IPC)

* Pipes
* Message queues
* Communication between user-space processes
* IPC available through system calls

### 👤 Authentication & Security

* Root and normal-user login
* UID and GID
* SHA-256 password hashing
* File ownership
* Unix-style file permissions
* `chmod`, `chown`, and `stat`

### 💾 Memory Management

* Paging
* Two-level x86 page tables
* 4 KB pages
* Kernel/user memory protection
* Dynamic kernel heap
* `kmalloc()` / `kfree()`
* First-fit free-list allocator

### ⏱️ CPU Scheduling

The OS supports four scheduling modes:

| Scheduler       | Description                                                     |
| --------------- | --------------------------------------------------------------- |
| **Round Robin** | Gives processes a fixed time quantum in circular order          |
| **Priority**    | Runs higher-priority processes first with aging                 |
| **FCFS**        | Processes execute according to arrival/creation order           |
| **AI-Adaptive** | Uses AI to classify workload and dynamically select the quantum |

---

# 🤖 AI Integration

The main AI component of this project is an **AI-Assisted Round Robin scheduler**.

Traditional Round Robin gives every process the same fixed time quantum. However, different processes have different workloads.

For example:

* Interactive processes need quick responses.
* CPU-intensive processes need longer CPU time.
* I/O-intensive processes frequently wait for I/O.
* Background processes generally need less immediate CPU attention.

Our AI scheduler uses a **Decision Tree classifier** to classify the recent behavior of a process.

### Workload Classes

The classifier identifies:

* Short Running
* Interactive
* I/O Intensive
* Long Running
* CPU Intensive
* Background

Based on the predicted workload, the scheduler selects an appropriate time quantum.

### Example

| Workload      |  Quantum |
| ------------- | -------: |
| Short Running |  3 ticks |
| Interactive   |  4 ticks |
| I/O Intensive |  6 ticks |
| Background    |  6 ticks |
| CPU Intensive | 12 ticks |
| Long Running  | 12 ticks |

### How It Works

```text
Process
   ↓
Collect Scheduling Telemetry
   ↓
AI Decision Tree Classifier
   ↓
Identify Workload Type
   ↓
Select Time Quantum
   ↓
AI-Adaptive Round Robin
   ↓
CPU Scheduling
```

The original ML model was developed using **scikit-learn**. Since the kernel cannot directly load Python/scikit-learn `.pkl` models, the learned decision-tree logic was converted into **plain C++** and integrated directly into the kernel.

This allows the AI scheduler to operate inside the OS rather than being only an external simulation.

---

# 📊 AI Scheduler vs Normal Round Robin

### Normal Round Robin

```text
P1 → P2 → P3 → P1 → P2 → P3
```

Every process receives the same fixed quantum.

### AI-Adaptive Round Robin

```text
P1 → P2 → P3
↓     ↓     ↓
4     12     6
ticks ticks  ticks
```

The execution order remains Round Robin, but the **quantum is adapted according to workload classification**.

> **Normal RR = fixed quantum**
> **AI-Adaptive RR = workload-based quantum**

---

# 🖥️ System Architecture

```text
                    User Applications
                           │
                    Ring 3 User Mode
                           │
                    System Calls
                           │
        ┌──────────────────┴──────────────────┐
        │              Kernel                 │
        │                                     │
        │  Process Management                 │
        │  IPC                                │
        │  Filesystem                         │
        │  Memory Management                  │
        │  Authentication                     │
        │                                     │
        │          CPU Scheduler              │
        │              │                      │
        │      ┌───────┼────────┐             │
        │      │       │        │             │
        │     RR   Priority    FCFS           │
        │              │                      │
        │        AI-Adaptive RR               │
        │              │                      │
        │       Decision Tree                 │
        └──────────────┼──────────────────────┘
                       │
                  Hardware / CPU
```

---

# ⏱️ Timer & Interrupt System

The OS uses the **Programmable Interval Timer (PIT)** to generate regular timer interrupts.

The timer operates at **100 Hz**, generating 100 timer interrupts per second.

The timer is used for:

* CPU scheduling
* Context switching
* Process timi
