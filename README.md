32-BIT PROTECTED-MODE OPERATING SYSTEM: ADVANCED + AI EDITION
=============================================================

An educational 32-bit x86 operating system written in C/C++ and NASM assembly. It covers the core of a Unix-like kernel: protected mode, ring 3 user processes, paging, a heap allocator, process management with fork(), IPC, authentication and file permissions, a persistent filesystem, an interactive shell, and four CPU scheduling policies, including an AI-adaptive Round Robin scheduler that picks each process's time quantum from a classified workload type.

FEATURES
--------

Protected mode and privilege levels
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

- 32-bit protected mode with GDT and IDT
- Ring 0 kernel mode and Ring 3 user mode
- TSS-based Ring 3 → Ring 0 stack switch on interrupts and system calls
- int 0x80 system-call ABI usable from Ring 3
- Kernel/user memory protection

Memory management
~~~~~~~~~~~~~~~~~

- Paging with two-level x86 page tables and 4 KB pages
- Per-process user mappings (code window and private stack page)
- Dynamic kernel heap: kmalloc() / kfree() with a first-fit free-list allocator

Process management
~~~~~~~~~~~~~~~~~~

- Process creation and termination, fork(), exit()
- Context switching and preemptive scheduling driven by a 100 Hz timer
- Process monitor commands (ps, info, metrics)

Inter-process communication
~~~~~~~~~~~~~~~~~~~~~~~~~~~

- Pipes and message queues, available to Ring 3 processes through system calls

Authentication and security
~~~~~~~~~~~~~~~~~~~~~~~~~~~

- Login prompt with root and normal-user accounts, UID and GID
- SHA-256 password hashing
- File ownership and Unix-style permissions (chmod, chown, stat) enforced on open, read and write

Filesystem
~~~~~~~~~~

- Persistent block filesystem with directories on an ATA PIO disk driver

CPU scheduling
~~~~~~~~~~~~~~

| Scheduler       | Description                                                         |
| Round Robin | Fixed time quantum, processes served in circular order              |
| Priority    | Higher-priority processes run first, with aging to prevent starvation |
| FCFS        | Processes run in arrival order                                      |
| AI-Adaptive | Classifies each process's workload and picks its quantum dynamically |

Switch modes from the shell, for example scheduler ai.

AI-ADAPTIVE SCHEDULING
----------------------

Traditional Round Robin gives every process the same quantum, even though workloads differ: interactive tasks want fast turnaround on short bursts, CPU-bound tasks want longer slices. The AI-adaptive mode keeps Round Robin ordering but sets each task's quantum from its recent behaviour.

    Process
       ↓
    Scheduling telemetry (CPU time, wait, priority, run-queue length)
       ↓
    Decision-tree workload classifier   (ai/classifier.cpp)
       ↓
    Workload class → base quantum       (ai/adaptive_quantum.cpp)
       ↓
    Load and priority rules, clamp      (ai/dynamic_rr.cpp)
       ↓
    Time quantum used by Round Robin

The classifier identifies six workload classes, each mapped to a base quantum:

| Workload      | Base quantum |
| Short Running | 3 ticks      |
| Interactive   | 4 ticks      |
| I/O Intensive | 6 ticks      |
| Background    | 6 ticks      |
| CPU Intensive | 12 ticks     |
| Long Running  | 12 ticks     |

ai/dynamic_rr.cpp then adjusts the base quantum: it shrinks when the run queue is long (6 or more runnable tasks), doubles when a task is the only runnable one, nudges it up or down by priority, and clamps the result to 2–20 ticks. Type drr in the shell to see each task's last decision, the reason for it, and real kernel counters (CPU ticks, ready ticks, response time, dispatches).

The classifier is a decision tree whose logic was derived from a scikit-learn model fit offline on synthetic scheduler telemetry. A kernel cannot load a Python model, so the tree's decision rules were ported by hand into plain C++ and run inside the kernel. The training data and training script are not part of this repository, so treat the classifier as a fixed, hand-ported rule table.

Evaluation: simulated results
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

casestudy/ contains a tick-level simulator (drr_sim.cpp) that runs standard Round Robin (quantum 5 and 10) and Dynamic RR on identical process sets, using the same ai/dynamic_rr.cpp decision code as the kernel: 7 scenarios × 30 seeds. In this first, untuned version, Dynamic RR improves response time but worsens average waiting and turnaround time in mixed workloads. These are simulation results, not kernel measurements. See casestudy/CASESTUDY_REPORT.md and casestudy/results/.

    cd casestudy && make results

SYSTEM ARCHITECTURE
-------------------

                     User Applications
                            │
                     Ring 3 User Mode
                            │
                      System Calls (int 0x80)
                            │
         ┌──────────────────┴──────────────────┐
         │                Kernel               │
         │                                     │
         │   Process Management    IPC         │
         │   Filesystem            Memory Mgmt │
         │   Authentication        Drivers     │
         │                                     │
         │            CPU Scheduler            │
         │     RR · Priority · FCFS · AI-RR    │
         └──────────────────┬──────────────────┘
                            │
                      Hardware / CPU

TIMER AND INTERRUPTS
--------------------

The Programmable Interval Timer (PIT) runs at 100 Hz, generating 100 timer interrupts per second. The timer drives CPU scheduling, context switching and process timing. The legacy PIC is remapped and IRQs are dispatched through the IDT.

REPOSITORY LAYOUT
-----------------

    boot/          Multiboot entry point
    arch/x86/      GDT, TSS, IDT, PIC, interrupts, paging, user-mode entry
    drivers/       VGA, keyboard, timer, serial
    mm/            kmalloc/kfree heap allocator
    libc/          Freestanding string and memory helpers
    scheduler/     Task structure, RR / Priority / FCFS / AI-Adaptive scheduler, context switch
    ai/            Workload classifier, adaptive quantum, dynamic RR decision logic
    process/       Ring 3 process control blocks, fork, exec glue
    syscall/       int 0x80 system-call ABI (kernel side and user-mode wrappers)
    ipc/           Pipes and message queues
    user/          Accounts, SHA-256 auth, Unix-style permissions, built-in user apps
    filesystem/    Block filesystem and ATA PIO disk driver
    monitor/       ps / info / metrics
    shell/         Interactive shell
    demo/          Background demo tasks (toggle with `demo on` / `demo off`)
    casestudy/     Standard RR vs Dynamic RR simulator, results and report
    kernel_main.cpp, linker.ld, grub.cfg, Makefile

BUILD AND RUN
-------------

Install the toolchain (Debian/Ubuntu):

    sudo apt-get install nasm xorriso grub-pc-bin grub-common mtools \
                         gcc-multilib g++-multilib qemu-system-x86

Build and boot in QEMU:

    make          # builds kernel.elf, os.iso and a 16 MB disk image
    make run      # boots in QEMU

Log in with root / root or user / user (educational defaults), then type help for the command list. To use a cross-compiler instead of your system g++ -m32:

    make CXX=i686-elf-g++

Other targets: make run-headless (no display, serial on stdio), make debug (QEMU waits for GDB on port 1234), make clean, make distclean.

TRY IT
------

    scheduler ai           # switch to the AI-adaptive scheduler
    ps                     # live WORKLOAD and QTM columns
    drr                    # last decision and reason for each task

    run init               # Ring 3 process; a blocked write to kernel memory shows protection works
    run pipe_test          # fork() + pipe: child blocks until the parent writes
    run msg_sender         # fork() + message queue
    run scheduler_test     # Ring 3 process yielding under the scheduler
    run permission_test    # file-permission check as a normal user

NOTABLE BUGS FOUND AND FIXED
----------------------------

These were found by booting the kernel in QEMU and driving the shell interactively:

1. User stack outside the paged range: the Ring 3 stack was mapped at 0xBFFFE000, but paging only covered the low 32 MB, so the first user process faulted on its own stack. Moved to 0x01FFF000.
2. fork() gave the child a stale register frame: the parent's frame was copied before being refreshed, so a first-time child resumed at EIP=0. Fixed by saving the live frame first.
3. Pipe reference counts ignored fork(): closing one end in either process zeroed the count for both, so writes failed and the child's read loop hung the system. Fixed by incrementing counts for each inherited descriptor.
4. chmod / chown crashed the kernel: the shell's fsCmd() reserved an 8 KB stack buffer on every call, consuming the entire 8 KB task stack. Fixed by making the buffer static and raising TASK_STACK_SIZE to 16 KB.

KNOWN LIMITATIONS
-----------------

- While any Ring 3 process is alive, Ring 3 scheduling takes over the timer interrupt, so a Ring 3 process that never exits hangs the whole system. All bundled test apps terminate correctly.
- No SMP, no ELF loader for arbitrary user binaries (the Ring 3 apps are built into the kernel image), and no network stack.
- The Dynamic RR logic and the drr command were added after the last full QEMU test pass, and the Dynamic RR results come from simulation only. A kernel-level benchmark comparing scheduler rr and scheduler ai is future work.
