## Chapter 1(Operating system interfaces)

- Xv6 time-shares processes: it transparently switches the available CPUs among the set of processes waiting to execute.

### Processes and memory 

![Screenshot 2023-07-12 at 10.32.02](/Users/ferriem/Library/Application Support/typora-user-images/Screenshot 2023-07-12 at 10.32.02.png)

`exec()` : child process run specific system call. Notice fork and exec are not combined in a single call.

`sbrk()` grow its data memory by n.

### I/O and File descripors

We can easily imply `cat` with fork, `fork` create a copy of its parent, and has its own memory space, also they **shared offse**t, it means when both call `write`, they can keep a consistent offset instead of recover the other just like using the fault offset.

Each underlying file is shared between parent and child.

The `dup` system call duplicates an existing file descriptor, returning a new one that refers to the same underlying I/O object. Both **file share an offset**.

### Pipes

- create a pipe 

  ```c++
  int p[2];// p[0] refers to the read end of the pipe, p[1] on the contrary.
  pipe(p);
  ```

- connection between pipe(especially between parent and child process)

  ```c++
  if(fd == 0)//create a child process
  {
  	close(0);//release 0 in file descriptor stack to make it available
  	dup(p[0]);//Now 0 is the lowest fd available, redirect stdin to p[0]
  	//redirect
  	
  	close(p[0]);//means any attempts to read from p[0] will fail, but the input will still redirected to p[0]
  	close(p[1]);
  	//child process didn't need to do with pipe
  	exec("/bin/ls", argv);
  }else{
  	close(p[0]);
  	write(p[1], "hello world\n", 12);
  	close(p[1]);
  }
  close(p[0]);
  close(p[1]);
  //ensure the pipe were released completely to avoid leek.
  ```

### File system

- ```c++
  chdir("path")//cd
  open("path", O_CREATE|O_WRONLY); //create a new file
  mknod("path", 1, 1);//create a device file, associated with major and minor device numbers
  ```

  When a process later opens a device file, the kernel diverts `read` and `write` calls to **kernel device** implementation instead of passing them to the file system.

- underlying file called *inode* can have multiple names called *links*,



