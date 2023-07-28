## Lab System call

### Using gdb

You can check [this](https://pdos.csail.mit.edu/6.828/2022/labs/gdb.html) to learn more about GDB.

Run `make qemu-gdb` in one terminal under the `xv6-lab-2022` directory. And run `riscv64-unknown-elf-gdb`(macos) on another terminal in the same path.

You may encounter the problem below:

```sh
warning: File ".../.gdbinit" auto-loading has been declined
...
To completely disable this security protection add
		set auto-load safe-path /
lint to your configuration file "..."
```

What you need to do is `vim ~/.gdbinit` and add "auto-load safe-path" (as the terminal said).

```sh
prompt >riscv64-unknown-elf-gdb
...
0x0000000000001000 in ?? () //this shows that gdb successfully opened.
(gdb) b syscall
Breakpoint 1 at 0x80001fe4: file kernel/syscall.c, line 133.
(gdb) c
Continuing.
[Switching to Thread 1.2]

Thread 2 hit Breakpoint 1, syscall () at kernel/syscall.c:133
133	{
(gdb) layout src //Shows where gdb is in
...
(gdb) backtrace
#0  syscall () at kernel/syscall.c:133
#1  0x0000000080001d18 in usertrap () at kernel/trap.c:67  //Q1
#2  0x0505050505050505 in ?? ()
(gdb) n
...
(gdb) n
...
(gdb) p /x *p
$1 = (lock = {lock = 0x0, name = 0x80008178, cpu = 0x0}, state = 0x4, chan = 0x0, killed = 0x0, xstate = 0x0, pid = 0x1, parent = 0x0, kstack = 0x3fffffd000, sz = 0x1000, pagetable = 0x87f3000, trapframe = 0x87f74000, context = {ra = 0x 80001464, sp = 0x3fffffde80, s0 = 0x3fffffdeb0, s1 = 0x80008d10, s2 = 0x800088e0, s3 = 0x1, s4 = 0x8000eb98, s5 =0x3, s6 = 0x800199b0, s7 = 0x1, s8 = 0x80019ad8, s9 = 0x4, s10 = 0x0,s11 = 0x0}, ofile = {0x0 <repeats 16 times>}, name = {0x69, 0x6e, 0x69, 0x74, 0x63, 0x6f, 0x64, 0x65, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0}}
(gdb)p p->trapframe->a7
$2 = 7 //Q2
(gdb) p /x $sstatus
$3 = 0x200000022
```

- Q1. Looking at the backtrace output, which function called `syscall`?

We can check the *kernel/trap.c:67* to know that function `usertrap()` calls `syscall()`.

- Q2. What is the value of `p->trapframe->a7` and what does that value represent? (Hint: look `user/initcode.S`, the first user program xv6 starts.)

The value is 7,  in "*user/initcode.S*" we can find `li a7, SYS_exect`at start,

- Q3. What was the previous mode that the CPU was in?

See [RISC-V privileged instructions](https://github.com/riscv/riscv-isa-manual/releases/download/Priv-v1.12/riscv-privileged-20211203.pdf) (page 64). We can see SPP (8 bit) is 0 now, corresponding to user mode.



Replace the statement `num = p->trapframe->a7;` with `num = * (int * ) 0;`(*kernel/sycall.c:137*)

```sh
prompt >make qemu
...
xv6 kernel is booting

hart 2 starting
hart 1 starting
scause 0x000000000000000d
sepc=0x0000000080001ff8 stval=0x0000000000000000
panic: kerneltrap

```

- Q4. Write down the assembly instruction the kernel is panicing at. Which register corresponds to the varialable `num`?

We can see `  80001ff8:	00002683          	lw	a3,0(zero) # 0 <_entry-0x80000000>` in *kernel/kernel.asm*. Obviously the answear is `lw a3,0(zero)`and a3

- Q5. Why does the kernel crash? Hint: look at figure 3-3 in the text; is address 0 mapped in the kernel address space? Is that confirmed by the value in `scause` above?

```sh
(gdb) n
press Ctrl+c
(gdb) p $scause
$1 = 13
```

Then search for the [RISC-V privileged instructions](https://github.com/riscv/riscv-isa-manual/releases/download/Priv-v1.12/riscv-privileged-20211203.pdf) (page 71) to know it raises a `load page fault`.  Look at [book](https://pdos.csail.mit.edu/6.828/2022/xv6/book-riscv-rev1.pdf) (figure 3.3) shows that virtual address 0 didn't corresoond to any physical address.

- Q6. What is the name of the binary that was running when the kernel paniced? What is its process id (`pid`)?

```sh
(gdb) b syscall
Breakpoint 1 at 0x80001fe4: file kernel/syscall.c, line 133.
(gdb) c
Continuing.
[Switching to Thread 1.3]

Thread 3 hit Breakpoint 1, syscall () at kernel/syscall.c:133
133	{
(gdb) n
135	  struct proc *p = myproc();
(gdb) n
138	  num = * (int *) 0;
(gdb) p p->name
$1 = "initcode\000\000\000\000\000\000\000"
(gdb) p *p
$2 = {lock = {locked = 0, name = 0x80008178 "proc", cpu = 0x0},
  state = RUNNING, chan = 0x0, killed = 0, xstate = 0, pid = 1, parent = 0x0,
  kstack = 274877894656, sz = 4096, pagetable = 0x87f73000,
  trapframe = 0x87f74000, context = {ra = 2147488868, sp = 274877898368,
    s0 = 274877898416, s1 = 2147519760, s2 = 2147518688, s3 = 1,
    s4 = 2147543960, s5 = 3, s6 = 2147588528, s7 = 1, s8 = 2147588824, s9 = 4,
    s10 = 0, s11 = 0}, ofile = {0x0 <repeats 16 times>},
  cwd = 0x80016e20 <itable+24>, name = "initcode\000\000\000\000\000\000\000"}
(gdb)
```

The name is **initcode**, and the pid is **1**.

### System call tracing

- **Target**: To create a new `trace` system call to control tracing (similar to strace in Linux), 
- **int trace(int n)** :Using n in binary to determine which syscall should be traced (*kernel/syscall.h*) .

*user/trace.c* provide us the program with basic framework.

- At first, add `$U/_trace` toUPROGS in Makefile

- Add the prototype for system call to *user/user.h* `int trace(int);`

- Add a stub to *user/usys.pl* (which produce *user/usys.S*), `entry("trace");`

- Add a syscall number to *kernel/syscall.h* `#define SYS_trace 22`

- Add a `sys_trace()` function in *kernel/sysproc.c*, the argument mask should be claimed in *proc.h*, add `int mask;` in `enum procstate`.

  ```
  uint64
  sys_trace(void)
  {
    int mask;
    argint(0, &mask);
    myproc()->mask = mask;
    return 0;
  }
  ```

  

- Modify `fork()` (*kernel/proc.c*), just add `np->mask = p->mask;` to copy the mask information to child process.

- Modify the `syscall()` (kernel/syscall.c) to print trace output.

  Add `extern uint64 sys_trace(void);` and `[SYS_trace]	sys_trace,`to proper place. Add a name mapping to better print.

  ```
  static char *syscallnames[] = {
  [SYS_fork]    "fork",
  [SYS_exit]    "exit",
  [SYS_wait]    "wait",
  [SYS_pipe]    "pipe",
  [SYS_read]    "read",
  [SYS_kill]    "kill",
  [SYS_exec]    "exec",
  [SYS_fstat]   "fstat",
  [SYS_chdir]   "chdir",
  [SYS_dup]     "dup",
  [SYS_getpid]  "getpid",
  [SYS_sbrk]    "sbrk",
  [SYS_sleep]   "sleep",
  [SYS_uptime]  "uptime",
  [SYS_open]    "open",
  [SYS_write]   "write",
  [SYS_mknod]   "mknod",
  [SYS_unlink]  "unlink",
  [SYS_link]    "link",
  [SYS_mkdir]   "mkdir",
  [SYS_close]   "close",
  [SYS_trace]   "trace",
  };
  ```

  Then modify the `syscall()`, we should look at the "1" bit in mask to specify the output. `if(mask >> num & 0b1) output;` is the core of the implementation.

  ```
  if(num > 0 && num < NELEM(syscalls) && syscalls[num]) {
      // Use num to lookup the system call function for num, call it,
      // and store its return value in p->trapframe->a0
      p->trapframe->a0 = syscalls[num]();
      if((p->mask >> num) & 1){
        printf("%d %s: syscall %s -> %d\n",
                p->pid, p->name, syscallnames[num], p->trapframe->a0);
      }
    } else {
      printf("%d %s: unknown sys call %d\n",
              p->pid, p->name, num);
      p->trapframe->a0 = -1;
    }
  }
  ```

Run `make qemu`

```sh
prompt >make qemu
...
$trace 32 grep hello README (32 = 2^5) only print read syscall
3:syscall read -> 1023
3:syscall read -> 966
3:syscall read -> 70
3:syscall read -> 0
$trace 2 usertests forkforkfork
usertests starting
...
ALL TESTS PASSED
$
```

### Sysinfo

- **Target**: add a sysinfo syscall, collecting information about the running system.

Add `$U/_sysinfotest` to UPROGS in Makefile.

Add the prototype for system call to *user/user.h* `struct sysinfo;` and `int sysinfo(struct sysinfo *)`.

Add a stub to *user/usys.pl* (which produce *user/usys.S*), `entry("sysinfo");`.

Add a syscall number to *kernel/syscall.h* `#define SYS_sysinfo 23.

The sysinfo call tells the **free memory** and **UNUSED process**. We need to write the funtion compute the two main argument.

- In *kernel/kalloc.c*

  ```
  uint64
  get_freemem(void)
  {
    uint64 count = 0;
    struct run *r;
    r = kmem.freelist;
    while(r){
      count += PGSIZE;
      r = r->next;
    }
    return count;
  }
  ```

- In *kernel/proc.c*

  ```
  uint64
  get_freeproc(void)
  {
    uint64 count = 0;
    struct proc *p;
    for(p = proc; p < &proc[NPROC]; p++){
      if(p->state != UNUSED)
        count++;
    }
    return count;
  }
  ```

- After write down the two fundemantal function, we need to declare them in *kernel/def.h*

  ```
  uint64          get_freemem(void);
  uint64          get_freeproc(void);
  ```

- And then in *sysproc.c*, we need to write the crux and use the two functions above.(The function `copyout()` can be learned in *kernel/file.c:97*)

  ```
  uint64
  sys_sysinfo()
  {
    uint64 si_addr;
    argaddr(0, &si_addr);
    struct sysinfo info;
    info.freemem = get_freemem();
    info.nproc = get_freeproc();
    if(copyout(myproc()->pagetable, si_addr, (char *)&info, sizeof(info)) < 0)
      return -1;
    return 0;
  }
  ```

- At last, sysinfo is applied in syscall.c

  ```
  extern uint64 sys_sysinfo(void);
  static uint64 (*syscalls[])(void) = {
  ...
  [SYS_sysinfo] sys_sysinfo,
  };
  static char *syscallnames[] = {
  ...
  [SYS_sysinfo] "sysinfo",
  };
  ```

Now everything is completed.

Run `make qemu` in terminal.

```sh
prompt >make qemu
...
$ trace 32 grep hello README
3 grep: syscall read -> 1023
3 grep: syscall read -> 961
3 grep: syscall read -> 321
3 grep: syscall read -> 0
$ trace 2147483647 grep hello README
3 trace: syscall trace -> 0
3 grep: syscall exec -> 3
3 grep: syscall open -> 3
3 grep: syscall read -> 1023
3 grep: syscall read -> 961
3 grep: syscall read -> 321
3 grep: syscall read -> 0
3 grep: syscall close -> 0
$ trace 2 usertests forkforkfork
usertests starting
3 usertests: syscall fork -> 4
test forkforkfork: 3 usertests: syscall fork -> 5
5 usertests: syscall fork -> 6
6 usertests: syscall fork -> 7
6 usertests: syscall fork -> 8
...
OK
3 usertests: syscall fork -> 66
ALL TESTS PASSED
$ sysinfotest
sysinfotest: start
sysinfotest: OK
$ 
```

Everything fine inside qemu, but about `make grade`, it may raise signal 15, I have no idea about it.

