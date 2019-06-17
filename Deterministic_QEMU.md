# Deterministic QEMU:

## Intro

### What is QEMU: 
QEMU is an open source emulator and virtual machine that allows a full-system guest operating system to be run as a normal process on a host operating system. 

### How do we know that QEMU is not deterministic: 
When a QEMU process (Guest OS) is being run. if we observe the sequence of instructions being executed by the guest, we notice that the sequence is not exactly identical in two different executions from the same initial state. A straightforward reason which explains this behavior is asynchronous interrupts. For example a keyboard interrupt during one execution sequence might cause the guest to execute extra instructions for interrupt handling and thus would be different from an execution sequence where the corresponding interrupt is absent. However even in the absence of such events, any two such sequences are not exactly the same.

### Goals
The goal of this document is therefore to provide a solution for running QEMU deterministically. Such a deterministic QEMU can be very useful when debugging applications on the guest since it would guarantee that any error is reproduced exactly at the same place in every execution. one very popular usage of this is developing an OS or debugging heavily multithreaded programs.



### What causes non deterministic behaviour:
generally speaking, there are three factors that make a process run in a non-deterministic manner:
* wall clock time
* thread context switches
* 

#### - Wall clock time:
In the original qemu architecture, there are timer queues associated with each type of clock that qemu maintains. These queues contain the time at which the cpu thread needs to be interrupted and related events be processed by the IO thread. Based on the earliest timer deadline across all such queues, a host timer is set up to send a SIGALRM signal once the earliest deadline is reached.

#### Thread context switches
Here again, synchronization between the IO thread the CPU thread is established by using host-clock time. as host kernel manages the scheduler and in general, modern schedulers have lots of parameters and most of are there for performance reasons,

## Some General Knowledge

### - General Knowledge:
QEMU is an event-driven program with a main loop that invokes callback functions when file descriptors or timers become ready. Callbacks become hard to manage when multiple steps are needed as part of a single high-level operation.

### QEMU Internals

#### Overall architecture and threading model
Running a guest involves executing guest code, handling timers, processing I/O, and responding to monitor commands. Doing all these things at once requires an architecture capable of mediating resources in a safe way without pausing guest execution if a disk I/O or monitor command takes a long time to complete. There are two popular architectures for programs that need to respond to events from multiple sources:

Parallel architecture splits work into processes or threads that can execute simultaneously. I will call this threaded architecture.

Event-driven architecture reacts to events by running a main loop that dispatches to event handlers. This is commonly implemented using the select(2) or poll(2) family of system calls to wait on multiple file descriptors.

QEMU actually uses a hybrid architecture that combines event-driven programming with threads. It makes sense to do this because an event loop cannot take advantage of multiple cores since it only has a single thread of execution. In addition, sometimes it is simpler to write a dedicated thread to offload one specific task rather than integrate it into an event-driven architecture. Nevertheless, the core of QEMU is event-driven and most code executes in that environment.

#### The event-driven core of QEMU

An event-driven architecture is centered around the event loop which dispatches events to handler functions. QEMU's main event loop is main_loop_wait() and it performs the following tasks:

Waits for file descriptors to become readable or writable. File descriptors play a critical role because files, sockets, pipes, and various other resources are all file descriptors. File descriptors can be added using qemu_set_fd_handler().
Runs expired timers. Timers can be added using qemu_mod_timer().
Runs bottom-halves (BHs), which are like timers that expire immediately. BHs are used to avoid reentrancy and overflowing the call stack. BHs can be added using qemu_bh_schedule().

When a file descriptor becomes ready, a timer expires, or a BH is scheduled, the event loop invokes a callback that responds to the event. Callbacks have two simple rules about their environment:
No other core code is executing at the same time so synchronization is not necessary. Callbacks execute sequentially and atomically with respect to other core code. There is only one thread of control executing core code at any given time.

No blocking system calls or long-running computations should be performed. Since the event loop waits for the callback to return before continuing with other events, it is important to avoid spending an unbounded amount of time in a callback. Breaking this rule causes the guest to pause and the monitor to become unresponsive.

This second rule is sometimes hard to honor and there is code in QEMU which blocks. In fact there is even a nested event loop in qemu_aio_wait() that waits on a subset of the events that the top-level event loop handles. Hopefully these violations will be removed in the future by restructuring the code. New code almost never has a legitimate reason to block and one solution is to use dedicated worker threads to offload long-running or blocking code.

#### Offloading specific tasks to worker threads

Although many I/O operations can be performed in a non-blocking fashion, there are system calls which have no non-blocking equivalent. Furthermore, sometimes long-running computations simply hog the CPU and are difficult to break up into callbacks. In these cases dedicated worker threads can be used to carefully move these tasks out of core QEMU.

One example user of worker threads is posix-aio-compat.c, an asynchronous file I/O implementation. When core QEMU issues an aio request it is placed on a queue. Worker threads take requests off the queue and execute them outside of core QEMU. They may perform blocking operations since they execute in their own threads and do not block the rest of QEMU. The implementation takes care to perform necessary synchronization and communication between worker threads and core QEMU.

Since the majority of core QEMU code is not thread-safe, worker threads cannot call into core QEMU code. Simple utilities like qemu_malloc() are thread-safe but that is the exception rather than the rule. This poses a problem for communicating worker thread events back to core QEMU.

When a worker thread needs to notify core QEMU, a pipe or a qemu_eventfd() file descriptor is added to the event loop. The worker thread can write to the file descriptor and the callback will be invoked by the event loop when the file descriptor becomes readable. In addition, a signal must be used to ensure that the event loop is able to run under all circumstances. This approach is used by posix-aio-compat.c and makes more sense (especially the use of signals) after understanding how guest code is executed.

#### Executing guest code

So far we have mainly looked at the event loop and its central role in QEMU. Equally as important is the ability to execute guest code, without which QEMU could respond to events but would not be very useful.

There are two mechanism for executing guest code: Tiny Code Generator (TCG) and KVM. TCG emulates the guest using dynamic binary translation, also known as Just-in-Time (JIT) compilation. KVM takes advantage of hardware virtualization extensions present in modern Intel and AMD CPUs for safely executing guest code directly on the host CPU. For the purposes of this post the actual techniques do not matter but what matters is that both TCG and KVM allow us to jump into guest code and execute it.

Jumping into guest code takes away our control of execution and gives control to the guest. While a thread is running guest code it cannot simultaneously be in the event loop because the guest has (safe) control of the CPU. Typically the amount of time spent in guest code is limited because reads and writes to emulated device registers and other exceptions cause us to leave the guest and give control back to QEMU. In extreme cases a guest can spend an unbounded amount of time without giving up control and this would make QEMU unresponsive.

In order to solve the problem of guest code hogging QEMU's thread of control signals are used to break out of the guest. A UNIX signal yanks control away from the current flow of execution and invokes a signal handler function. This allows QEMU to take steps to leave guest code and return to its main loop where the event loop can get a chance to process pending events.

The upshot of this is that new events may not be detected immediately if QEMU is currently in guest code. Most of the time QEMU eventually gets around to processing events but this additional latency is a performance problem in itself. For this reason timers, I/O completion, and notifications from worker threads to core QEMU use signals to ensure that the event loop will be run immediately.

You might be wondering what the overall picture between the event loop and an SMP guest with multiple vcpus looks like. Now that the threading model and guest code has been covered we can discuss the overall architecture.

#### iothread and non-iothread architecture

The traditional architecture is a single QEMU thread that executes guest code and the event loop. This model is also known as non-iothread or !CONFIG_IOTHREAD and is the default when QEMU is built with ./configure && make. The QEMU thread executes guest code until an exception or signal yields back control. Then it runs one iteration of the event loop without blocking in select(2). Afterwards it dives back into guest code and repeats until QEMU is shut down.

If the guest is started with multiple vcpus using -smp 2, for example, no additional QEMU threads will be created. Instead the single QEMU thread multiplexes between two vcpus executing guest code and the event loop. Therefore non-iothread fails to exploit multicore hosts and can result in poor performance for SMP guests.

Note that despite there being only one QEMU thread there may be zero or more worker threads. These threads may be temporarily or permanent. Remember that they perform specialized tasks and do not execute guest code or process events. I wanted to emphasise this because it is easy to be confused by worker threads when monitoring the host and interpret them as vcpu threads. Remember that non-iothread only ever has one QEMU thread.

The newer architecture is one QEMU thread per vcpu plus a dedicated event loop thread. This model is known as iothread or CONFIG_IOTHREAD and can be enabled with ./configure --enable-io-thread at build time. Each vcpu thread can execute guest code in parallel, offering true SMP support, while the iothread runs the event loop. The rule that core QEMU code never runs simultaneously is maintained through a global mutex that synchronizes core QEMU code across the vcpus and iothread. Most of the time vcpus will be executing guest code and do not need to hold the global mutex. Most of the time the iothread is blocked in select(2) and does not need to hold the global mutex.

Note that TCG is not thread-safe so even under the iothread model it multiplexes vcpus across a single QEMU thread. Only KVM can take advantage of per-vcpu threads.

### Coroutines: 

Coroutines are cooperatively scheduled threads of control. There is no preemption timer that switches between coroutines periodically, instead switching between coroutines is always explicit. Coroutines run until termination or an explicit yield.

Cooperative scheduling makes writing coroutine code simpler than writing multi-threaded code. Only one coroutine executes at a time and it cannot be interrupted. In many cases this eliminates the need for locks since other coroutines cannot interfere while the current coroutine is running.

In other words, coroutines allow multiple tasks to be executed concurrently in a disciplined fashion.

## - Implementation Steps

### - Switch from Kernel-Level Threading to User-Level Threading
Modern OS kernels provide sophisticated schedulers and usually among all the factors that they take into consideration while doing their scheduling algorithm, deterministic thread-context switches is not one of them. and this is one of the root causes of non-deterministic behaviour. to be able to manage the thread-context switches in a deterministic manner, we need to design a scheduler which provides non-preemptive priority-based scheduling for multiple threads of execution (aka multithreading) inside our event-driven applications. All threads should run in the same address space of the application, but each thread can have it's own individual program-counter, run-time stack, signal mask and errno variable. The thread scheduling itself should be done in a cooperative way, i.e., the threads should be managed by a priority- and event-based non-preemptive scheduler. The intention is that this way one can achieve better run-time performance than with preemptive scheduling. The event facility allows threads to wait until various types of events occur, including pending I/O on filedescriptors, asynchronous signals, elapsed timers, pending I/O on message ports, thread and process termination, and even customized callback functions.

### - Using Virtual ticks instead of wall clock time
Another aspect of deterministic execution relies on how QEMU handles events and timed filedescriptors. to remove any external dependency here, we need to switch from using wall clock time to QEMU's Virtual Ticks, which provides a reference to a clock that is based on instructions being executed.

### - Provide an alternative to QEMU's usage of GCC TLS:
coming soon

## Options

### Compiling with PTH
In order to run QEMU in a deterministic manner, one needs to install GNU PTH and turn the '--enable-pth' flag on when configuring QEMU:

./configure --target-list=aarch64-softmmu --enable-pth
make -j8

### Running with PTH

./qemu-system--aarch64 ..... -d loop X
where X is the number of vCPU loops that QEMU's vCPU thread performs before being forced to yield back to the PTH scheduler.

### - Evaluation
Simple: run qemu twice and log results. see if results match.
