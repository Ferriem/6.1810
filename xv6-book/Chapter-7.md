## Chapter 7(Scheduling)

​	This chapter mainly explain how xv6 achieves multiplexing.

### Multiplexing

​	Xv6 multiplexes by switching each CPU from one process to anotherr in two situations. First xv6's `sleep` and `wakeup` nechanism switch. Second, xv6 periodically forces a switch to copy with processes that compute for long periods without sleeping.

### Code: Context switching

​	Examine the mechanics of switching between a kernel thread and a scheduler thread.

​	The function `swtch` performs the aces and restores for a kernel thread switch. `swtch` doesn't directly know about threads, it just saves and restores sets of 32 RISC-V registers, called *contexts*. Each context is contained in a `struct context` (*kernel/proc.h:2*), itself contained in a process's `struct proc` or a CPU's `struct cpu.swtch` takes two arguments: `struct context *old` and `struct context ~new`.

​	We saw in Chapter 4 that at the end of an interrupts is that `usertrap` calls `yield`. `yield` in turn calls `sched`, which calls `swtch` to save the current context in `p->context` and switch to the scheduler context previously saved in `cpu->context`(*kernel/proc.c*).

​	`swtch`(*kernel/swtch/S:3*) saves only callee-saved registers; `swtch`knows the offset of each register's field in `struct context`, it doesn't save the `pc`. Instead, `swtch` saves the `ra` register, holding the **return address** from which `swtch` was called. Return to `ra` and return on the new thread's stack since that's where the restored `sp` points.

### Code: Scheduling

​	The scheduler exists in the form of a special thread per CPU, each running the `scheduler` function choosing which process to run next.

​	The only place a kernel thread gives up its CPU is in `sched`. and it always switches to the same location in `scheduler`(switchess to some kernel thread that previously called `sched`).

### Code: mycpu and myproc

​	Xv6 maintains a `struct cpu` for each CPU(*kernel/proc.h:22*), which records the process currently running on that CPU, saved registers for the CPU's scheduler thread, and the count returns a pointer to the current CPU's `struct cpu`.. `kernel_hartid` in `struct trapframe`makes asking easily.

​	`cpuid` and `mycpu` disable interrupts, and only enable them after they finish using the returned `struct cpu`.

​	`myproc`(*kernel/proc.c:83*) returns the `struct proc` pointer for the process that is running on the current CPU. It invokes `mycpu`, fetches the current process pointer(`c->proc`) out of the `struct cpu`.

### Sleep and wakeup

​	Build a high-level synchronization mechanism called a *semaphore*. A semaphore maintains a count and provides two operations. The "V" operation for producer increments the count. The "P" operation for the consumer waits until the count is non-zero, and then decrement it and returns.

​	Xv6's `sleep`(*kernel/proc.c:536*) and `wakeup`(*kernel/proc.c:567*), `sleep` mark the current process at SLEEPING and then call `sched` to release the CPU; `wakeup` looks for a process sleeping on the given wait channel and marks it as RUNNABLE.

​	`sleep` acquires `p->lock`. Now the process going to sleep holds both `p->lock` and `lk`. Holding `lk` ensure no other process could start a call to `wakeup(chan)`. Now `sleep` holds `p->lock`, it is sage to release `lk`.

​	`wakeup(chan)` acquires the `p->lock`, loops over the process table to find a process in state SLEEPING with a matching `chan`, it changes the process's state to RUNNABLE.

### Code: Wait, exit and kill

​	The way that xv6 records the child's demise until `wait` observes it is for `exit` to put the caller into the ZOMBIE state, where it stays until the parent's `wait` notices it, changes the child's state to UNUSED. If the parent exits before the child, the parent gives the child to the `init` process. A challenge is to avoid races and deadlock between simulataneous parent and child `wait` and `exit`, as well as the simultaneous `exit` and `exit`.

​	`wait` starts by acquiring `wait_lock`(kernel/proc.c:391), then `wait` scans the process table to find a child in ZOMBIE state, it frees that child's recourses and its `proc` structure, copies the child's exit status to the address supplied to `wait`. and returns the child's PID. `pp->lock` to avoid deadlock.

​	`exit`(kernel/proc.c:347) records the exit status, free some resources, calls `reparent` to give its children to the `init` process, wakes up the parent in case it is in `wait`, marks the caller as a zombie, and permanently yields the CPU. Holding `wait_lock` and `p->lock` during this sequence. `wait_lock` prevents a parent in `wait` from losing the wakeup. `p->lock` prevents a parent in `wait` seeing the child is in state `ZOMBIE` before the child has finally called `swtch.exit`/

`kill` sets the victim's `p->killed` and, if it is sleeping, wakes it up. Eventually the victim will enter or leave the kernel at which point code in `usertrap` will call `exit` if `p-killed` is set.

### Process Locking

​	`p->lock` must be held while reading or writing any of the following `struct proc` fields: `p->state`, `p->chan`, `p->killed`, `p->xstate`, and `p->pid`.

​	Here's the full set of things that `p->lock` does

- Along with `p->state`, it prevents races in allocating `proc[]` slots for new process.
- It conceals a process from view while it is being created or destroyed.
- It prevents a parent's `wait` from collecting a process that has set its state to ZOMBIE but has not yet yielded the CPU
- It prevents another core’s scheduler from deciding to run a yielding process after it sets its state to RUNNABLE but before it finishes swtch.

- It ensures that only one core’s scheduler decides to run a RUNNABLE processes.
- It prevents a timer interrupt from causing a process to yield while it is in swtch.
- Along with the condition lock, it helps prevent wakeup from overlooking a process that is calling sleep but has not finished yielding the CPU.
- It prevents the victim process of kill from exiting and perhaps being re-allocated between kill’s check of p->pid and setting p->killed.
- It makes kill’s check and write of p->state atomic.

​	The `p->parent` field is protected by the global lock `wait_lock` rather tahn by `p->lock`.