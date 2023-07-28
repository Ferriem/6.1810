## Lab lock

### Memory allocator

- **Target**: modify kalloc to make each CPU has its own freelist.

Following the hints to modify the `struct kmem`

```
struct {
  struct spinlock lock;
  struct run *freelist;
} kmem[NCPU];
```

And then with the `kinit`

```
void
kinit()
{
  for(int i = 0; i < NCPU; i++){
    initlock(&kmem[i].lock, "kmem");
  }
  freerange(end, (void*)PHYSTOP);
}
```

- Hint: Imply `cpuid` and `push_off` and `pop_off` to get the current core number.

For `kfree` and `kalloc`, easily get the core number and modify the former code.(Simply change `kmem` to `kmem[i]`).

```
void
kfree(void *pa)
{
  struct run *r;

  if(((uint64)pa % PGSIZE) != 0 || (char*)pa < end || (uint64)pa >= PHYSTOP)
    panic("kfree");

  // Fill with junk to catch dangling refs.
  memset(pa, 1, PGSIZE);

  r = (struct run*)pa;
  push_off();
  int i = cpuid();
  pop_off();
  acquire(&kmem[i].lock);
  r->next = kmem[i].freelist;
  kmem[i].freelist = r;
  release(&kmem[i].lock);
}

void *
kalloc(void)
{
  struct run *r;
  push_off();
  int index = cpuid();
  pop_off();
  for(int i = 0; i < NCPU; i++){
    acquire(&kmem[index].lock);
    r = kmem[index].freelist;
    if(r)
      kmem[index].freelist = r->next;
    release(&kmem[index].lock);
    if(r)
      break;
    index = (index + 1) % NCPU;
  }

  if(r)
    memset((char*)r, 5, PGSIZE); // fill with junk

  return (void*)r;
}
```

```sh
prompt >./grade-lab-lock kalloctests
...
== Test running kalloctest == (66.3s)
== Test   kalloctest: test1 ==
  kalloctest: test1: OK
== Test   kalloctest: test2 ==
  kalloctest: test2: OK
== Test   kalloctest: test3 ==
  kalloctest: test3: OK
== Test kalloctest: sbrkmuch == kalloctest: sbrkmuch: OK (11.8s)
```

### Buffer cache

- **Target**: imply a more fragile lock to fasten the buffer cache

​	Following the Hints, add `#define NBUCKET      13` to *param.h*.

​	To make the buffer cache faster, I apply a hash to map `blockno`and each need a lock to avoid race and deadlock. Don't implement LRU (buf->prev). You can change the value of `mul` to change the size of buffer.

```
#define mul 2
struct {
  struct spinlock lock;
  struct buf buf[NBUCKET*mul];
  struct spinlock bucket[NBUCKET];
  struct buf head[NBUCKET];
} bcache;
```

- Init the buffer

Each bucket allocate two nodes, which mapping to the element of buf array.

```
binit(void)
{
  struct buf *b;

  initlock(&bcache.lock, "bcache");

  for (int i = 0; i < NBUCKET; ++i) {
    initlock(&bcache.bucket[i], "bcache.bucket");
    bcache.head[i].next = 0;
    b = &bcache.head[i];
    for(int j = i * mul; j < i * mul + mul; j++){
      initsleeplock(&bcache.buf[j].lock, "buffer");
      bcache.buf[j].next = b->next;
      b->next = &bcache.buf[j];
      b = b->next;
    }
  }
}
```

- Look for the buffer

​	Get the mapping number to `blockno`, traverse the bucket to find the cache. If found, inc the block's ref, release the lock of bucket, acquire the lock of block, return the block.

​	If not found, search if there are free block in the bucket. If find the free block, allocate it; if not, get other free block in other bucket. When search the other bucket, we need to assign bcache lock to avoid deadlock.

- Release the buffer

Sub the ref.

```
void
binit(void)
{
  struct buf *b;

  initlock(&bcache.lock, "bcache");

  for (int i = 0; i < NBUCKET; ++i) {
    initlock(&bcache.bucket[i], "bcache.bucket");
    bcache.head[i].next = 0;
    b = &bcache.head[i];
    for(int j = i * 2; j < i * 2 + 2; j++){
      initsleeplock(&bcache.buf[j].lock, "buffer");
      bcache.buf[j].next = b->next;
      b->next = &bcache.buf[j];
      b = b->next;
    }
  }
}

static struct buf*
bget(uint dev, uint blockno)
{
  struct buf *b;

  int index = blockno % NBUCKET;

  acquire(&bcache.bucket[index]);
  for (b = bcache.head[index].next; b; b = b->next) {
    if (b->blockno == blockno && b->dev == dev) {
      b->refcnt++;
      release(&bcache.bucket[index]);
      acquiresleep(&b->lock);
      return b;
    }
  }

  for (b = bcache.head[index].next; b; b = b->next) {
    if (b->refcnt == 0) {
      b->blockno = blockno;
      b->dev = dev;
      b->valid = 0;
      b->refcnt = 1;
      release(&bcache.bucket[index]);
      acquiresleep(&b->lock);
      return b;
    }
  }
  release(&bcache.bucket[index]);

  acquire(&bcache.lock);
  for (int i = 0; i < NBUCKET; ++i) {
    if (i != index) {
      struct buf *pre;
      acquire(&bcache.bucket[i]);
      for (pre = &bcache.head[i], b = bcache.head[i].next; b; pre = b, b = b->next) {
        if (b->refcnt == 0) {
          pre->next = b->next;
          release(&bcache.bucket[i]);

          acquire(&bcache.bucket[index]);
          b->next = bcache.head[index].next;
          bcache.head[index].next = b;
          b->blockno = blockno;
          b->dev = dev;
          b->valid = 0;
          b->refcnt = 1;
          release(&bcache.bucket[index]);
          acquiresleep(&b->lock);
          release(&bcache.lock);
          return b;
        }
      }
      release(&bcache.bucket[i]);
    }
  }
  release(&bcache.lock);

  panic("bget: no buffers");
}

void
brelse(struct buf *b)
{
  if(!holdingsleep(&b->lock))
    panic("brelse");

  int index = b->blockno % NBUCKET;

  releasesleep(&b->lock);

  acquire(&bcache.bucket[index]);
  b->refcnt--;
  release(&bcache.bucket[index]);
}

void
bpin(struct buf *b) {
  int index = b->blockno % NBUCKET;

  acquire(&bcache.bucket[index]);
  b->refcnt++;
  release(&bcache.bucket[index]);
}

void
bunpin(struct buf *b) {
  int index = b->blockno % NBUCKET;

  acquire(&bcache.bucket[index]);
  b->refcnt--;
  release(&bcache.bucket[index]);
}
```

`bpin` and `bunpin` modify the `refcnt` which refer to the bucket, we need use the bucket lock to maintain it.

```sh
prompt > make grade
...
== Test running kalloctest ==
$ make qemu-gdb
(73.3s)
== Test   kalloctest: test1 ==
  kalloctest: test1: OK
== Test   kalloctest: test2 ==
  kalloctest: test2: OK
== Test   kalloctest: test3 ==
  kalloctest: test3: OK
== Test kalloctest: sbrkmuch ==
$ make qemu-gdb
kalloctest: sbrkmuch: OK (11.7s)
== Test running bcachetest ==
$ make qemu-gdb
(14.3s)
== Test   bcachetest: test0 ==
  bcachetest: test0: OK
== Test   bcachetest: test1 ==
  bcachetest: test1: OK
== Test usertests ==
$ make qemu-gdb
usertests: OK (56.8s)
== Test time ==
time: OK
Score: 80/80
```

