## Lab Multithreading

### Switching between threads

- **Target**: implement context switch by completing `thread_create` and `thread_schedule` and `uthread_switch.S`.

​	Read Chapter 7 of the [book](https://pdos.csail.mit.edu/6.828/2022/xv6/book-riscv-rev3.pdf), register `ra` stores the return address and `sp` stores the stack pointer. Add a `struct context` to store the state, I totally copy rhe `struct context` in *kernel/proc.h*.

​	And then when `thread_schedule()` occurs, we should perform context switch here, as we cheak the *Makefile*, `thread_schedule()` just compile the function in *uthread_switch.S* to restore the registers and load new registers. Also I copied the code in *kernel/swtch.S*.

In *user/uthread.c*

```
struct context {
  uint64 ra;
  uint64 sp;

  // callee-saved
  uint64 s0;
  uint64 s1;
  uint64 s2;
  uint64 s3;
  uint64 s4;
  uint64 s5;
  uint64 s6;
  uint64 s7;
  uint64 s8;
  uint64 s9;
  uint64 s10;
  uint64 s11;
};
void 
thread_create(void (*func)())
{
  struct thread *t;

  for (t = all_thread; t < all_thread + MAX_THREAD; t++) {
    if (t->state == FREE) break;
  }
  t->state = RUNNABLE;
  // YOUR CODE HERE
  memset(t->stack, 0, STACK_SIZE);
  memset(&t->context, 0, sizeof(struct context));
  t->context.sp = (uint64)t->stack + STACK_SIZE; 
  t->context.ra = (uint64)func; //restores the callee address.
  return ;
}
void 
thread_schedule(void)
{
...
// YOUR CODE HERE
	thread_switch((uint64)&t->context, (uint64)&next_thread->context);
...
}
```

In *uthread_switch.S*

```
thread_switch:
	/* YOUR CODE HERE */
	sd ra, 0(a0)
        sd sp, 8(a0)
        sd s0, 16(a0)
        sd s1, 24(a0)
        sd s2, 32(a0)
        sd s3, 40(a0)
        sd s4, 48(a0)
        sd s5, 56(a0)
        sd s6, 64(a0)
        sd s7, 72(a0)
        sd s8, 80(a0)
        sd s9, 88(a0)
        sd s10, 96(a0)
        sd s11, 104(a0)

        ld ra, 0(a1)
        ld sp, 8(a1)
        ld s0, 16(a1)
        ld s1, 24(a1)
        ld s2, 32(a1)
        ld s3, 40(a1)
        ld s4, 48(a1)
        ld s5, 56(a1)
        ld s6, 64(a1)
        ld s7, 72(a1)
        ld s8, 80(a1)
        ld s9, 88(a1)
        ld s10, 96(a1)
        ld s11, 104(a1)
	ret    /* return to ra */
```

```sh
prompt > ./grade-lab-thread uthread
...
== Test uthread == uthread: OK (3.9s)
```

### Using threads

- **Target**: implement lock in *notxv7/ph.c* to avoid key missing in multithreads.

​	Analyze the code, when insert the key to table, if two threads insert in a same place in nearly the same time, one will cover another. Now add a lock to it can solve the problem.

*ph.c*

```
...
pthread_mutex_t lock;
...
static 
void put(int key, int value)
{
  pthread_mutex_lock(&lock);
  int i = key % NBUCKET;

  // is the key already present?
  struct entry *e = 0;
  for (e = table[i]; e != 0; e = e->next) {
    if (e->key == key)
      break;
  }
  if(e){
    // update the existing key.
    e->value = value;
  } else {
    // the new is new.

    insert(key, value, &table[i], table[i]);
    
  }
  pthread_mutex_unlock(&lock);
}
...
int
main(int argc, char *argv[]){
	...
	pthread_mutex_init(&lock, NULL);
	...
}
```

```sh
prompt > make ph
prompt > ./ph 2
100000 puts, 2.629 seconds, 38044 puts/second
1: 0 keys missing
0: 0 keys missing
200000 gets, 2.554 seconds, 78308 gets/second
prompt > ./grade-lab-thread ph_safe
== Test ph_safe == make: `ph' is up to date.
ph_safe: OK (4.4s)
```

​	You can imply the lock just around `insert()`, it seems faster than get the lock in the beginning, but in my test they are done with almost the same time.

### Barrier

- **Target**: implement a `barrier`The desired behavior is that each thread blocks in `barrier()` until all `nthreads` of them have called `barrier()`.

```
static void 
barrier()
{
  pthread_mutex_lock(&bstate.barrier_mutex);
  bstate.nthread++;
 
  if(bstate.nthread == nthread){
    bstate.nthread = 0;
    bstate.round++;
    pthread_cond_broadcast(&bstate.barrier_cond);
  } else {
    pthread_cond_wait(&bstate.barrier_cond, &bstate.barrier_mutex);
  }
  pthread_mutex_unlock(&bstate.barrier_mutex);
}
```

```sh
prompt > make barrier
prompt > ./barrier 2
OK; passed
prompt > ./grade-lab-thread barrier
== Test barrier == make: `barrier' is up to date.
barrier: OK (1.8s)
```

