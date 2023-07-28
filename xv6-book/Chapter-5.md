## Chapter 5(Interrupts and device drivers)

​	In xv6, `devintr`(*kernel/trap.c:178*) handle a device raising an interrupt.

​	Many device drivers execute code in two contexts: a *top half* that runs in a process's kernel thread, and a *bottom half* that executes at interrupt time. 	

### Code: Console input

​	The console driver (*kernel/console.c*) is a simple illustration of driver structure. The console driver accepts characters typed by a human, **via *UART* serial-port** hardware attached to the RISC-V. When tou type input to xv6 in QEMU, your keystrokes are deliverd to xv6 by way of QEMU's simulated UART hardware.

​	There are some physical address that RISC-V hardware connectcs to the UART device, so that loads and stores interact with the device hardware rather than RAM. The memory-mapped addresses for the UART start at 0x10000000, or `UART0`(*kernel/menlayout.h:21*). There are a handful of UART control registers. Their offset from `UART0` are defined in *kernel/uart.c:22*. 

​	Xv6's `main` calls `concoleinit`(kernel/concole.c:182) to initialize the UART hardware. 

​	The xv6 shell reads from the console by way of a fd opened by *user/init.c:19*. Calls to the `read` system call to `consoleread` (*kernel/console.c:80*). `consoleread` waits fir input to arrive and be buffered in `cons.buf`, copies the input to user space and returns to the user process.

​	When the user types a character, the UART hardware asks the RISC-V to raise an interrupt to active xv6's trap handler which calls `devintr`. Map `scause` register to know it's an external device. Ask PLIC to tell which device interrupted. If it was the UART, `devintr` calls `uartintr`.

​	`uartintr` (*kernel/uart.c:176*) reads waiting input characters and hands them to `consoleintr`. `uartintr` doesn't wait, `consoleintr` accumulate input in `cons.buf` until a whole line arrives. When a newline arrives, `consoleintr` wakes up `consoleread`.

​	Once woken, `consoleread` will observe a full line in `cons.buf`, copy it yo user space and return to user space.

### Code: Console output

​	A `write` system call connected to console eventually arrives at `uartputc`(*kernel/uart.c:87*). The device driver maintains an output buffer(`uart-tx-buf`), `uartputc` appends each character to the buffer(no need to wait). Calls `uartstart` to start the device transmitting. The only situation in which `uartputc` waits is if the buffer is full.

​	Each time UART finishes sending a byte, it generates an interrupt. `uartintr` calls `uartstart`, checking whether the device really finished sending, and hands the device the next buffered output character.

### Concurrency in drives

​	Three concurrency dangers

- two processes on different CPUs might call consoleread at the same time.
- the hardware might ask a CPU to deliver a console (really UART) interrupt while that CPU is already executing inside `consoleread`.
- the hardware might deliver a console interrupt on a different CPU while consoleread is executing.

​	Chapter 6 explores these problems.

### Timer interrupts

​	The `yield` calls in `usertrap` and `kerneltrap` cause switching.

​	Code executed in machine mode in `start.c`, sets up receive timer interrupts, scratch area, `mtvec` to `timervec`.