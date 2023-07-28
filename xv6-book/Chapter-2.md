## Chapter 2

An operating system must fulfill **multiplexing, isolation, interaction**.

### Abstracting physical resources

- To achieve strong isolation between applications, the Unix applications interact with storage only through the file system's *open, read, write, close*. 

- Strong isolation requires a hard boundary between applications and operating system.

- RISC-V has three mode in which the CPU can execute instructions: *machine mode, supervisor mode and user mode*.

### Kernel organization

- Monolithic kernels: the entire operating system resides in the kernel.

- Microkernels: minimize the amount of operating system code that runs in supervisor mode.

- Linux has a monolithic kernel.

- Xv6 is implemented as a **monolithic** kernel, like most Unix operating systems. The inter-module interfaces are defined in *kernel//def.h*

### Process overview

- Xv6 uses page tables to give each process its own address space. Page table translates a *virtual address* to a *physical address*.
- Xv6 kernel maintains many pieces of state for each process, *kernel/proc.h:85*

### Starting xv6

​	When the RISC-V computer powers on, it initializes itself and runs a boot loader which is stored in read-only memory and load the kernel into memory. In machine mode, the CPU executes xv6 starting at `_entry`(kernel/entry.S:7). 

​	The loader loads the xv6 kernel into memory at physical address 0x80000000. Smaller contains I/O devices.

​	`_entry` set up a stack. Xv6 declares space for an initial stack, in the file `start.c`(kernel/start.c:11). Loading the stack pointer `sp` with the address `stack0+4096`, the top of stack.

​	RISC-V provides the instruction `mret` switches to supervisor mode. Sets previous privilege mode to supervisor in the register `mstatus`. Writing the `main`'s address into register `mepc`, delegates all interrupts and exceptions to supervisor mode.

​	Before jumping into supervisor mode, `start` programs the clock chip to generate timer interrupts.

​	After `main`(kernel/main.c:11) initializes several devices and subsystems, it creates the first process `userinit`(kernel/proc.c:233), loads the number for the `exec` system c;a;.

​	The kernel uses the number in register `a` 7 in `syscall`(kernel/syscall.c:132) to call the desired system call. The system call table(kernel/syscall.c:107) maps SYS_EXec to sys_exec.

​	Once the kernel completed `exec`, it returns to user space in the /init process. `init`(user/init.c:15) creates a new console device file if needed.

