## Lab file-system

### Large files

- **Target**: Increase the block number from 268 to 65803(256 * 256 + 256 + 11)

​	The first 11 elements of `ip->addrs[]` should be direct blocks; the 12th should be a singly-indirect block (just like the current one); the 13th should be your new doubly-indirect block.

- First at *fs.h* to modify the invariant number

  ```c++
  #define NDIRECT 11
  #define NINDIRECT (BSIZE / sizeof(uint))
  #define DOUBLEINDIRECT (NINDIRECT * NINDIRECT)
  #define MAXFILE (NDIRECT + NINDIRECT + DOUBLEINDIRECT)
  ```

  Because we modify NDIRECT from 12 to 11, the related delaration of `addrs[]` in `struct inode`(*file.h*) and `struct dinode`(*fs.h*) have to change.

  ```c++
  struct inode {
  	...
    uint addrs[NDIRECT+2];
  };
  struct dinode {
  	...
    uint addrs[NDIRECT+2];   // Data block addresses
  };
  ```

- Then we need to modify `bmap`(*fs.c*). We can imitate the former code. And modify `itrunc` to truncate the double indirect block.

```c++
static uint
bmap(struct inode *ip, uint bn)
{
  uint addr, *a, *a2;
  struct buf *bp, *bp2;

  if(bn < NDIRECT){
    if((addr = ip->addrs[bn]) == 0){
      addr = balloc(ip->dev);
      if(addr == 0)
        return 0;
      ip->addrs[bn] = addr;
    }
    return addr;
  }
  bn -= NDIRECT;

  if(bn < NINDIRECT){
    // Load indirect block, allocating if necessary.
    if((addr = ip->addrs[NDIRECT]) == 0){
      addr = balloc(ip->dev);
      if(addr == 0)
        return 0;
      ip->addrs[NDIRECT] = addr;
    }
    bp = bread(ip->dev, addr);
    a = (uint*)bp->data;
    if((addr = a[bn]) == 0){
      addr = balloc(ip->dev);
      if(addr){
        a[bn] = addr;
        log_write(bp);
      }
    }
    brelse(bp);
    return addr;
  }
  
  //add here
  bn -= NINDIRECT;

  if(bn < DOUBLEINDIRECT){
    // Load double indirect block, allocating if necessary.
    if((addr = ip->addrs[NDIRECT+1]) == 0){
      addr = balloc(ip->dev);
      if(addr == 0)
        return 0;
      ip->addrs[NDIRECT+1] = addr;
    }
    bp = bread(ip->dev, addr);
    a = (uint*)bp->data;
    if((addr = a[bn / NINDIRECT]) == 0){
      addr = balloc(ip->dev);
      if(addr == 0){ 
        return 0;
      }
      a[bn / NINDIRECT] = addr;
      log_write(bp);
    }
    brelse(bp);
    bp2 = bread(ip->dev, addr);
    a2 = (uint*)bp2->data;
    if((addr = a2[bn % NINDIRECT]) == 0){
      addr = balloc(ip->dev);
      if(addr){
        a2[bn % NINDIRECT] = addr;
        log_write(bp2);
      }
    }
    brelse(bp2);
    return addr;
  }
  //end add

  panic("bmap: out of range");
}

void
itrunc(struct inode *ip)
{
  int i, j;
  struct buf *bp, *bp2;
  uint *a, *a2;

  for(i = 0; i < NDIRECT; i++){
    if(ip->addrs[i]){
      bfree(ip->dev, ip->addrs[i]);
      ip->addrs[i] = 0;
    }
  }

  if(ip->addrs[NDIRECT]){
    bp = bread(ip->dev, ip->addrs[NDIRECT]);
    a = (uint*)bp->data;
    for(j = 0; j < NINDIRECT; j++){
      if(a[j])
        bfree(ip->dev, a[j]);
    }
    brelse(bp);
    bfree(ip->dev, ip->addrs[NDIRECT]);
    ip->addrs[NDIRECT] = 0;
  }
	
	//add here
  if(ip->addrs[NDIRECT + 1]){
    bp = bread(ip->dev, ip->addrs[NDIRECT + 1]);
    a = (uint*)bp->data;
    for(i = 0; i < NINDIRECT; i++){
      if(a[i]){
        bp2 = bread(ip->dev, a[i]);
        a2 = (uint*)bp2->data;
        for(j = 0; j < NINDIRECT; j++){
          if(a2[j])
            bfree(ip->dev, a2[j]);
        }
        brelse(bp2);
        bfree(ip->dev, a[i]);
      }
    }
    brelse(bp);
    bfree(ip->dev, ip->addrs[NDIRECT + 1]);
    ip->addrs[NDIRECT + 1] = 0;
  }
  
	//end add
  ip->size = 0;
  iupdate(ip);
}
```

```sh
prompt > ./lab-grade-fs bigfile
...
== Test running bigfile == running bigfile: OK (74.8s)
```

### Symbolic links

- **Target**: implement `symlink(char *target, char *path)`.

- First create a new syscall number for symlink.

  ```c++
  //in user/user.h
  int symlink(const char*, const char*);
  
  //in user/usys.pl
  entry("symlink");
  
  //in Makefile
  $U/_symlinktest\
  
  //in kernel/syscall.h
  #define SYS_symlink 22
  
  //in kernel/syscall.c
  extern uint64 sys_symlink(void);
  ...
  [SYS_symlink] sys_symlink,
  ```

- Add `#define T_SYMLINK 4` in *kernel/stat.h*

- Add `#define O_NOFOLLOW 0x800` in *kernel/fcntl.h*

- Implement the `symlink` system call with the function `sys_symlink` in *kernel/sysfile.c*, look at `link` as a guide.

  ```c++
  uint64
  sys_symlink(void)
  {
    char target[MAXPATH], path[MAXPATH];
    struct inode* ip;
    if(argstr(0, target, MAXPATH) < 0 || argstr(1, path, MAXPATH) < 0)
      return -1;
    if(namei(path) != 0)
      return -1;
    begin_op();
    if((ip = create(path, T_SYMLINK, 0, 0)) == 0){
      end_op();
      return -1;
    }
    if((writei(ip, 0, (uint64)target, 0, strlen(target))) < 0){
      ip->nlink = 0;
      iupdate(ip);
      iunlockput(ip);
      end_op();
      return -1;
    }
    iunlockput(ip);
    end_op();
    return 0;
  }
  ```

- Modify `open` system call. If the linked file is also a symbolic link, you must recursively follow it until a non-link file is reached. If the links form a cycle, you must return an error code. 

  ```c++
  uint64
  sys_open(void)
  {
    char path[MAXPATH];
    int fd, omode;
    struct file *f;
    struct inode *ip;
    int n;
  
    argint(1, &omode);
    if((n = argstr(0, path, MAXPATH)) < 0)
      return -1;
  
    begin_op();
  
    if(omode & O_CREATE){
      ip = create(path, T_FILE, 0, 0);
      if(ip == 0){
        end_op();
        return -1;
      }
    } else {
      if((ip = namei(path)) == 0){
        end_op();
        return -1;
      }
      ilock(ip);
      if(ip->type == T_DIR && omode != O_RDONLY){
        iunlockput(ip);
        end_op();
        return -1;
      }
    }
  
  //add here
    if(ip->type == T_SYMLINK){
      if(!(omode & O_NOFOLLOW)){
        int cycle = 0;
        while(1){
          cycle++;
          if(cycle == 10){
            iunlockput(ip);
            end_op();
            return -1;
          }
          if(readi(ip, 0, (uint64)path, 0, MAXPATH) < 0){
            iunlockput(ip);
            end_op();
            return -1;
          }
          iunlockput(ip);
          if((ip = namei(path)) == 0){
            end_op();
            return -1;
          }
          ilock(ip);
          if(ip->type != T_SYMLINK){
            break;
          }
        }
      }
    }
    
  //end add
  
    if(ip->type == T_DEVICE && (ip->major < 0 || ip->major >= NDEV)){
  	...
  }
  ```

```sh
prompt > make grade
== Test running bigfile ==
$ make qemu-gdb
running bigfile: OK (119.9s)
== Test running symlinktest ==
$ make qemu-gdb
(0.6s)
== Test   symlinktest: symlinks ==
  symlinktest: symlinks: OK
== Test   symlinktest: concurrent symlinks ==
  symlinktest: concurrent symlinks: OK
== Test usertests ==
$ make qemu-gdb
usertests: OK (172.2s)
== Test time ==
time: OK
Score: 100/100
```

