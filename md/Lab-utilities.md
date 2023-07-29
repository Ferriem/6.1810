## Lab Utilities

### Sleep

- **Target**: Implement sleep for xv6.

*user/echo.c* tells the way to obtain the command-line arguments passed to a program.

In *user/user.h* `int sleep(int)` tells the defination of sleep.

What we need to do is write a *sleep.c* program under *user* director

```c++
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

int
main(int argc, char *argv[])
{
    if(argc != 2){
        fprintf(2, "usage: sleep ticks\n");
    }
    int time = atoi(*++argv);
    if(sleep(time) != 0){
        fprintf(2, "sleep error\n");
    }
    exit(0);
}
```

Then we need to do with the Makefile, simply add `$U/_sleep\` under `UPROGS=\`

Run `make qemu` in terminal then excute sleep.

```sh
prompt> make qemu
...
init: starting sh
$ sleep 10
(nothing happened for a little while)
$ (press control + a, x to exit)
Run make grade to check
prompt> ./grade-lab-util sleep
make: `kernel/kernel' is up to date.
== Test sleep, no arguments == sleep, no arguments: OK (1.0s)
== Test sleep, returns == sleep, returns: OK (1.0s)
== Test sleep, makes syscall == sleep, makes syscall: OK (1.0s)
prompt >
```

### Pingpong(easy)

- **Target**: build interprocess connection through *pipe*. The parent send a byte to child, the child print "...", write the bytes on the pipe to the parent, the parent read the byte and print "...".

All we should do can finished in file */user/pingpong.c*:

```c++
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

int
main(int arg, char *argv[])
{
    int p[2]; // pipe;
    if(pipe(p) == -1) //create a pipe
    {
        fprintf(2, "pipe error\n");
    }
    if(fork() == 0)//create child process
    {
        close(p[1]);
        char buf[1];
        read(p[0], buf, 1); //receive the byte from parent
        fprintf(0, "%d: received ping\n", getpid());
        write(p[1], buf, 1); // write the byte to parent
        close(p[0]);
    }else{
        close(p[0]);
        write(p[1], "1", 1); // send a byte to child.
        close(p[1]);
        wait(0); //wait the child to finish
        char buf[1];
        read(p[0], buf, 1); //read the byte from child
        fprintf(0, "%d: received pong\n", getpid());
        close(p[0]);
    }
    exit(0);
}
```

Also add `$U/_pingpong\` under `UPROGS=\`

Run `make qemu` in terminal: 

```sh
prompt> make qemu
...
init: starting sh
$ pingpong
4: received ping
3: received pong
$
```

### Primes

- **Target**: Write a concurrent verison of prime sieve using pipes.
- Solution, we can create a **pipeline**, **read from the left pipe and write to the right pipe**, the first pipe can be the top ancestor, write all the number(begin with the smallest prime 2) to its child. The child receive from the parent and **we can promise the first number will be a prime**, than the child can print the prime and **use the prime number to sieve** the following numbe utill the no more read from the parent.

*/user/primes.c*: 

```c++
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

void new_pipe(int p[2]){
    close(p[1]);
    int prime;
    if(read(p[0], &prime, sizeof(int)) == 0){
        return ;// end of the read
    }
    fprintf(0, "prime %d\n", prime);//print out the prime
    int new_p[2]; 
    if(pipe(new_p) == -1)//create a new pipe in order to connect its own child.
    {
        fprintf(2, "pipe error\n");
    }
    if(fork() == 0)// child process
    {
        new_pipe(new_p);
    }else{
        close(new_p[0]);
        int n;
        while(read(p[0], &n, sizeof(int)) == sizeof(int)){
            if(n % prime) write(new_p[1], &n, sizeof(int)); //sieve
        }
        close(p[0]);
        close(new_p[1]);
        wait(0);
    }
}

int
main(int argc, char *argv[])
{
    int p[2]; 
    if(pipe(p) == -1) //create a pipe
    {
        fprintf(2, "pipe error\n");
    }
    if(fork() == 0) //create a child process
    {
        new_pipe(p);
    }else{
        close(p[0]);
        for(int i = 2; i <= 35; i++){
            write(p[1], &i, sizeof(int)); //write all the number into pipe.
        }
    }
    close(p[1]);
    wait(0);
    exit(1);
}
```

Add `$U/_primes`

```sh
prompt >make qemu
...
init starting sh
$ primes
prime 2
prime 3
prime 5
prime 7
prime 11
prime 13
prime 17
prime 19
prime 23
prime 29
prime 31
$
```

### Find

- **Target**: Write a UNIX find program to find all files in a directory tree with a specific name.

- Solution: check */user/ls* to get the comprehense of reading directories.

  ```c++
  struct dirent de; //dir entry
  struct stat st; //stat information
  int fd; //file descriptor
  int fstat(int fd, struct stat *st) //load the stat information of fd to st.
  struct st{
  	...
  }
  ```

*/user/find.c*:

```c++
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"
#include "kernel/fs.h"

void find(char *path, char *name)//recursive
{
    char buf[512], *p;
    int fd;
    struct dirent de;
    struct stat st;

    if((fd = open(path, 0)) < 0){
        fprintf(2, "ls: cannot open %s\n", path);
        return;
    }

    if(fstat(fd, &st) < 0){
        fprintf(2, "ls: cannot stat %s\n", path);
        close(fd);
        return;
    }

    switch(st.type){
    case T_DEVICE:
    case T_FILE: 
        fprintf(2, "find: %s is not a directory\n", path);
        break;
    case T_DIR:
        if(strlen(path) + 1 + DIRSIZ + 1 > sizeof buf){
        printf("ls: path too long\n");
        break;
        }
        strcpy(buf, path);
        p = buf+strlen(buf);
        *p++ = '/';
        while(read(fd, &de, sizeof(de)) == sizeof(de)){
            if(de.inum == 0)
                continue;
            if(strcmp(de.name, ".") == 0 || strcmp(de.name, "..") == 0)
                continue;
            memmove(p, de.name, DIRSIZ);
            p[DIRSIZ] = 0;
            if(stat(buf, &st) < 0){
                printf("ls: cannot stat %s\n", buf);
                continue;
            }
            if(st.type == T_DEVICE || st.type == T_FILE){
                if(strcmp(de.name, name) == 0){
                    printf("%s\n", buf);
                }
            }
            else find(buf, name);
        }
    }
}

int
main(int argc, char *argv[]){
    if(argc != 3){
        fprintf(2, "Usage: find <path> <name>\n");
        exit(1);
    }
    char *path = argv[1];
    char *name = argv[2];
    find(path, name);
    exit(0);
}
```

```sh
prompt >make qemu
...
init starting sh
$ echo > b
$ mkdir a
$ echo > a/b
$ mkdir a/aa
$ echo > a/aa/b
$ find . b
./b
./a/b
./a/aa/b
$
```

### Xargs 

- **Target** Write a UNIX xargs program.

eg. `echo a | xargs echo b` are equal to `echo b a` we need to transport the answear of the ahead program as the argument of follow program.

*user/xargs.c*

```c++
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"
#include "kernel/param.h"

int ptr;// number of args
char *args[MAXARG];// final args
char buf[512*MAXARG]; //restore argument from stdin

void xargs(){
    char *p = buf; //the beginning of buf.
    char ch;
    int count = 0;//num of lines.
    while(p && read(0, &ch, 1)){
        if(ch == '\n'){
            count++;
            *p++ = '\0';
        }
        else *p++ = ch;
    }
    //read from the stdin (also the argument from ahead program)
    if(read(0, &ch, 1)){
        fprintf(2, "xargs: input too long\n");
        exit(1);
    }

    if(count > MAXARG){
        fprintf(2, "xargs: too many arguments\n");
        exit(1);
    }

    p = buf;// now buf restores the args and we need to alocate it to *args.
    for(int i = 0; i < count; i++){
        args[ptr++] = p;
        p += strlen(p);
        p++;
    }
}

int 
main(int argc, char *argv[]){
    for(int i = 1; i < argc; i++){
        args[ptr++] = argv[i];
    }
    xargs();
    if(fork() == 0){
        if(exec(argv[1], args) < 0)//execute the instruction with args.
        {
            fprintf(2, "xargs: exec %s failed\n", argv[1]);
            exit(1);
        }
    }
    wait(0);
    
    exit(0);
}
```

```sh
prompt> make qemu
...
init starting sh
$ sh < xargstest.sh
$ $ $ $ $ $ hello
hello
hello
$ $
```





