## Lab mmap

- **Target**: Implement `mmap` and `munmap` to xv6.

```sh
prompt > man mmap
...
void *
mmap(void *addr, size_t len, int prot, int flags, int fd, off_t offset);

prompt > man munmap
...
int
munmap(void *addr, size_t len);
```

- First do some necessary preparations for the lab.

  ```c++
  //in user/usys.pl
  entry("mmap");
  entry("munmap");
  
  //in user/user.h
  void* mmap(void*, uint, int, int, int, uint);
  int munmap(void*, uint);
  
  //in Makefile
  $U/_mmaptest\
  
  //in kernel/sysproc.c
  uint64
  sys_mmap(void)
  {
  	return -1;
  }
  uint64
  sys_munmap(void)
  {
  	return 0;
  }
  
  //in kernel/syscall.h
  #define SYS_mmap   22
  #define SYS_munmap 23
  
  //in kernel/syscall.c
  extern uint64 sys_mmap(void);
  extern uint64 sys_munmap(void);
  ...
  SYS_mmap]    sys_mmap,
  [SYS_munmap]  sys_munmap,
  ```

- Then we need to design a structure conrrsponding to the VMA(virtual memory area), recording the address, length, permissions, file, etc. for a virtual memory range created by `mmap`. Add `\#define NVMA         16` in *kernel/param.h* I imply the 16 blocks with the allocation below.

  | 6 * 2PGSIZE  |
  | :----------: |
  | 4 * 4PGSIZE  |
  | 3 * 8PGSIZE  |
  | 2 * 16PGSIZE |
  | 1 * 32PGSIZE |

  Beside the information above, we also need a `pos` to record the vma's address which returned by `mmap`. And a `size` to record the size of virtual area.

  ```c++
  //in kernel/proc.h
  struct vma{
    struct file *file;
    uint64 addr;
    uint64 pos;
    uint length;
    uint size;
    int prot;
    int flag;
  };
  
  struct proc{
  ...
  	struct vma vma[NVMA]; //Virtual memory areas.
  }
  ```

  ![user-space](/Users/ferriem/Desktop/ferriem/6.1810/image/user-space.png)

  The picture inform us there are unused region below the trapframe area. So we can implement our VMA in the unused area.

  ```c++
  //in kernel/proc.c
  static struct proc*
  allocproc(void)
  {
  ...
  	uint64 va = MAXVA - 2 * PGSIZE;
    for(int i = 0; i < 6; i++){
      va -= PGSIZE * 2;
      p->vma[i].pos = va;
      p->vma[i].size = PGSIZE * 2;
    }
    for(int i = 6; i < 10; i++){
      va -= PGSIZE * 4;
      p->vma[i].pos = va;
      p->vma[i].size = PGSIZE * 4;
    }
    for(int i = 10; i < 13; i++){
      va -= PGSIZE * 8;
      p->vma[i].pos = va;
      p->vma[i].size = PGSIZE * 8;
    }
    for(int i = 13; i < 15; i++){
      va -= PGSIZE * 16;
      p->vma[i].pos = va;
      p->vma[i].size = PGSIZE * 16;
    }
    for(int i = 15; i < 16; i++){
      va -= PGSIZE * 32;
      p->vma[i].pos = va;
      p->vma[i].size = PGSIZE * 32;
    }
  }
  ```

- Now, let's work on the function `mmap`. As we do before, get the arguments with `argint`, `argaddr`. Generate a proc and file, and allocate the map. `mmap` should increase the file's reference count so that the structure doesn't disappear when the file is closed (hint: see `filedup`)

  ```c++
  uint64
  sys_mmap(void){
    uint64 addr;
    uint offset;
    uint length;
    int fd;
    int prot;
    int flag;
  
    argaddr(0, &addr);
    argint(1, (int *)&length);
    argint(2, &prot);
    argint(3, &flag);
    argint(4, &fd);
    argint(5, (int *) &offset);
  
    struct proc *p = myproc();
    struct file *f = p->ofile[fd];
  
    if(prot & PROT_READ){
      if(! f->readable){
        return -1;
      }
    }
    if(prot & PROT_WRITE){
      if(! f->writable && flag != MAP_PRIVATE){
        return -1;
      }
    }
    for(int i = 0; i < NVMA; i++){
      if(!p->vma[i].length && p->vma[i].size >= length){
        p->vma[i].file = f;
        p->vma[i].addr = p->vma[i].pos;
        p->vma[i].length = length;
        p->vma[i].prot = prot;
        p->vma[i].flag = flag;
        filedup(f);
        return p->vma[i].addr;
      }
    }
    return -1;
  }
  ```

- Now, if we run the `mmaptest`, it will raise `scause 0x000000000000000d`, check the figure below.![scause](/Users/ferriem/Desktop/ferriem/6.1810/image/scause.png)

  It is a page fault, we can handle it just like what we do in [Lab-cow](./Lab-cow.md), write a `vmpagefault` function in *krnel/trap.c* to handle it. (Allocate a page of physical memory, read 4096 bytes of the relevant file into that page, and map it into the user address space.)

  See *fcntl.h* and *riscv.h*

  ```c++
  #define PROT_NONE       0x0
  #define PROT_READ       0x1
  #define PROT_WRITE      0x2
  #define PROT_EXEC       0x4
  
  #define PTE_V (1L << 0) // valid
  #define PTE_R (1L << 1)
  #define PTE_W (1L << 2)
  #define PTE_X (1L << 3)
  #define PTE_U (1L << 4) // user can access
  ```

  We get the idea `PTE_R`, `PTE_W`, `PTE_X` can be represented by `port << 1`.

  ```c++
  int 
  vmapagefault(struct proc *p, uint64 va){
    int index;
    //get the fault VMA
    for(index = 0; index < NVMA; index++){
      if(p->vma[index].addr <= va && va < p->vma[index].addr + p->vma[index].length){
        break;
      }
    }
    
    if(index >= 16)return -1;
    
    //Allocate a physical page memory
    uint64 pa;
    if((pa = (uint64)kalloc()) == 0){
      return -1;
    }
    memset((void*)pa, 0, PGSIZE);
    
    //read the file
    ilock(p->vma[index].file->ip);
    if(readi(p->vma[index].file->ip, 0, pa, va - p->vma[index].pos, PGSIZE) < 0){
      kfree((void*)pa);
      iunlock(p->vma[index].file->ip);
      return -1;
    }
    iunlock(p->vma[index].file->ip);
    int perm = PTE_U | (p->vma[index].prot << 1);
    if(mappages(p->pagetable, va, PGSIZE, pa, perm) < 0){
      kfree((void*)pa);
      return -1;
    }
    return 0;
  }
  ```

  ```c++
  void
  usertrap(void)
  {
  	...
  	   syscall();
  	//add here
    } else if(r_scause() == 0xd){
      if(vmapagefault(p, r_stval()) < 0){
        setkilled(p);
      }
    //end add
    } else if((which_dev = devintr()) != 0){
    ...
  }
  ```

- And then we walk into `sys_munmap`, find the VMA for the address range and uncap the specified pages(use uvmunmap). If all pages were removed, decrement the reference count of `struct file`. If an unmapped page has been modified and the file is mapped `MAP_SHARED`, write the page back to the file. Look at `filewrite` for inspiration.

  ```c++
  uint64
  sys_munmap(void){
    uint64 addr;
    uint length;  
    argaddr(0, &addr);
    argint(1, (int *)&length);
  
    struct proc *p = myproc();
    int index;
    
  //find the VMA
    for(index = 0; index < NVMA; index++){
      if(p->vma[index].addr <= addr && addr < p->vma[index].addr + p->vma[index].length){
        break;
      }
    }
    
    
    if(index >= 16) return -1;
    if(length > p->vma[index].length) return -1;
    
    //write back
    if(p->vma[index].flag == MAP_SHARED){
      filewrite(p->vma[index].file, addr, length);
    }
    
    
    uint64 va;
    for(va = addr; va < addr + length; va += PGSIZE){
      pte_t *pte = walk(p->pagetable, va, 0);
      if(*pte & PTE_V){
        uvmunmap(p->pagetable, va, 1, 0);
      }
    }
    p->vma[index].addr = addr + length;
    p->vma[index].length -= length;
    if(p->vma[index].length == 0){
      fileclose(p->vma[index].file);
    }
    return 0;
  }
  ```

  Till now, we can pass all the test before `fork_test`.

- Modify `fork` in *kernel/proc.c*, ensure that the child has the same mapped regions as the parent. Increment the reference count for a `struct file`, it is OK to allocate a new physical page,.

  ```c++
  	safestrcpy(np->name, p->name, sizeof(p->name));
  	//add here	
  	for(int i = 0; i < NVMA; i++){
      np->vma[i] = p->vma[i];
      if(np->vma[i].length){
        filedup(np->vma[i].file);
      }
    }
    //end add
    pid = np->pid;
  ```

- And then `exit`, we need to unmap the process's mapped regions as if `munmap` is called, so we can just imitate the code in `munmap`.

  ```c++
  for(int i = 0; i < NVMA; i++){
      if(p->vma[i].length){
        if(p->vma[i].flag == MAP_SHARED){
          filewrite(p->vma[i].file, p->vma[i].addr, p->vma[i].length);
        }
        uint64 va;
        for(va = p->vma[i].addr; va < p->vma[i].addr + p->vma[i].length; va += PGSIZE){
          pte_t *pte = walk(p->pagetable, va, 0);
          if(*pte & PTE_V){
            uvmunmap(p->pagetable, va, 1, 0);
          }
        }
        fileclose(p->vma[i].file);
        p->vma[i].length = 0;
      }
    }
  ```

```sh
prompt >make grade
...
== Test running mmaptest ==
$ make qemu-gdb
(5.2s)
== Test   mmaptest: mmap f ==
  mmaptest: mmap f: OK
== Test   mmaptest: mmap private ==
  mmaptest: mmap private: OK
== Test   mmaptest: mmap read-only ==
  mmaptest: mmap read-only: OK
== Test   mmaptest: mmap read/write ==
  mmaptest: mmap read/write: OK
== Test   mmaptest: mmap dirty ==
  mmaptest: mmap dirty: OK
== Test   mmaptest: not-mapped unmap ==
  mmaptest: not-mapped unmap: OK
== Test   mmaptest: two files ==
  mmaptest: two files: OK
== Test   mmaptest: fork_test ==
  mmaptest: fork_test: OK
== Test usertests ==
$ make qemu-gdb
usertests: OK (57.9s)
== Test time ==
time: OK
Score: 140/140
```

