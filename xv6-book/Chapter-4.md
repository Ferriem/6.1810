## Chapter 4(Traps and system calls)

​	Xv6 handles all traps in the kernel.

### RISC-V trap machinary

​	*kernel/riscv.h* contains definition of control registers to handle traps:

- `stvec` (**Supervisor Trap Vector**)  The kernel writes the address of its trap handle here; the RISC-V jumps to the address in `stvec` to handle a trap.(the first handle instruments are sometimes called **vector**).
- `sepc` (**Supervisor Exception Program Counter**) save the `pc` here.(Since `stvec` will overwrite the `pc`). The `sret`(return from trap) instruction copies `sepc` to the `pc`.
- `scause`(**Supervisor cause**) the reason for the trap.
- `sscratch`avoid overwriting user registers before saving them.
- `sstatus` The **SIE bit** in `sstatus` controls whether device interrupts are enabled.(If the kernel clears SIE, the RISC-V will defer device interrupts until the kernel sets SIE) The **SPP bit** indicates whether a tarp came from user mode or a supervisor mode, and controls to what mode `sret`return.

​	When it needs to force a trap, the RISC-V hardware does the following. 

- If the trap is a device interrupt, and the `sstatus` SIE bit is clear, don't do and of the following.
- Disable interrupts by clearing the SIE bit in `sstatus`.

- Copy `pc` to `sepc`.
- Save the current mode in the SPP bit in `sstatus`.
- Set `scause` to reflect the trap's cause.
- Set the mode to supervisor.
- Copy `stvec` to `pc`.
- Start executing new `pc`.

​	CPU doesn't switch to page table and stack in kernel. Kernel software must perform these tasks.

### Traps from user space

- The high-level path of a trap from user space is `uservec`(*kernel/tarmpoline.S:32*), then `usertrap`(*kernel/trap.c:37*); when returning, `usertrapret`(*kernel/trap.c:90*) and then `userret`(*kernel/trampoline.S:101*).
- Both the user page table and the kernel page table must have a mapping for the handle pointed to by `stvec`.
- Xv6 satisfies these requirements using a `trampoline` page. The trampoline page  contains `uservec`. The trampoline page is mapped in every process'spage table at address `TRAMPOLINE`, which is at the top of the virtual address space. `TRAMPOLINE` is also mapped in kernel page table
- The code for the `uservec` trap handler is in trampoline.S(*kernel/trampoline.S:21*). When `uservec` starts, all 32 registers need to be saved somewhere in the memory in the form of the `sscratch` register. `csrw` instruction at the start of `uservec` saves `a0` in `sscratch`/
- a page of memory for a `trapframe` structure has space to save the 32 user register(*kernel/proc.h:43*). Xv6 maps each process's trapframe at virtual address `TRAPFRAME` in that process's user page table; `TRAPFRAME` is just below `TRAMPOLINE`.
- Thus `uservec` load address `TRAPFRAME` into `a0` and saves all the user registers there.
- `trapframe` contiams the address of the current process's kernel stack, the current CPU's hartid, the address of the `usertrap` function, and the address of the kernel page table.

- The job of `usertrap` is to determine the cause of the trap, process it,and return.(*kernel/trap.c:37*). It changes `stvec` so that a trap in the kernel will be handled by `kernelvec`. It saves `sepc` register. If the trap is a system call, `usertrap` calls `syscall` to handle it.
- The first step in returning to user space is the call to `usertrapret`(*kernel/trap.c:90*). Changing `stvec` to refer to `uservec`, preparing the trapframe field that `uservec`relies on, and setting `sepc` to previously saved user `pc`. At the end, `userrapret` calls `userret` on the trapoline page that is mapped in both user and kernel page tables.
- `userrt` passes a pointer to the process's user page table in a0(*kernel/trampoline.S:101*). `userret` switches `satp` to process's user page table.

### Code: Calling system calls

​	`initcode.S` places the arguments for `exec` in registers `a0` and `a1`, and oputs the system call number in `a7`. The `ecall` instruction traps into the kernel and causes `uservec`, `usertrap` and `syscall`.

​	`syscall`(*kernel/syscall.c:132*) retrieves the system call number from the saved `a7` in the trapframe and uses it to index into `syscalls`.

​	When `sys_exec` returns, `syscall` records its return value in `p->trapframe->a0`. 

### Code: System call arguments

- System call implementations in the kernel need to find the aruguments passed by user code. The kernel trap code saves user registers to the current process's trap frame. The kernel function `argint`, `argaddr` and `argfd` retrieve the n'th system call argument from the trap frame as  an integer, pointer or a file descriptor. They all call `argraw` to retrieve the appropriate saved user register(*kernel/syscall.c:34*).
- Some system calls pass pointers as arguments. The `exec` system call, for example, passes the kernel an array of pointers referring to srtring arguments in user space. (But the transfer can have a lot of problems).
- The kernel implements functions `fetchstr`(*kernel/syscall.c:25*). `fetchstr` calls `copyinstr` to retrieve string file-name arguments from user space.
- `copyinstr`(*kernel/vm.c:403*) copies up to `max` byte s to `dst` from virtual address `srcva` in the user page table `pagetable`. Since the `pagetable `is not the current page table, `copyinstr` uses `walkaddr` to look up `srcva` in `pagetable`. A similar function `copyout`, copies data from the kernel to a user-supplied address.

### Traps from kernel space

- `kernelvec` saves registers on the stack of the interrupted kernel thread.
- `kernelvec` jumps to `kerneltrap`(*kernel/trap.c:135*) after saving registers. There are two kinds of trap: device interrupts and exception. It calls `devintr`(*kernel/trap.c:178*) to check for and handle the former. If it isn't a device interrupt, it must be an exception, the kernel calls `panic` and stops executing.
- Things left are return, copies `sepc` to `pc`.(Some details  about `sstatus` and `yield` will be referred in subquent chapter). Now just know `yield` give other threadss a chance to run.

### Page-fault exceptions

​	If an exception happens in user space, the kernel kills the faulting process. If an exception happens in the kernel, the kernel panics.

​	Many kernels uses page faults to implement *copy-on-write*(COW) fork. 

​	Parent and child can safely share physical memory byu appropriate use of page-table permissions and page faults. The CPU **raises a *page-fault exception*** when a virtual address is used that has no mapping in the page table or `PTE_V` flag is clear, or a permission bits obid the operation being attempted. The `scause` register indicates the type of the page fault and the `stval` register contains the address that couldn't be translated.

​	The **basic plan** in COW fork is for the parent and child to initially share all physical pages, but for each to map them read-only. If either writes a given page, the RISC-V CPU raises a page-fault exception. And do some operations to keep consistency.

​	Copy-on-write makes `fork` faster, since the `fork` need not copy memory.

​	Another widely-used feature by calling ***lazy allocation***. When an application asks for more memory by calling `sbrk`, the kernel notes the increase in size, but does not allocate physical memory and does not create PTEs for the new range of virtial address. On a page fault on one of those new addresses, the kernel allocates a page of physical memory and maps it into the page table.

​	Yet another widely-used feature is ***demand pageing***.

​	The programs running on a computer may need more memory than the computer has RAM. To copy gracefully, the operating system may implement ***paging to disk***. The idea is to store only a fraction of user pages in RAM, and to store the rest on disk in a *paging area*.

​	

