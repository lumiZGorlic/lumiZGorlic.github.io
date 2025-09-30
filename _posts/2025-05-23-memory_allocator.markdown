---
layout: post
title:  "memory allocator"
date:   2025-05-23 01:39:55 +0100
categories: jekyll update
---

The task 'memory allocator' concerns the memory allocation (and also locks). What is it all about ? First off there's this global thingy.

{% highlight c %}
struct {
  struct spinlock lock;
  struct run *freelist;
} kmem;
{% endhighlight %}

It has a lock (at the end of the post I explain more about how locking works) and a pointer to something which looks like this

{% highlight c %}
struct run {
  struct run *next;
};
{% endhighlight %}

so we can say that freelist is a pointer to an object that itself is nothing more than a pointer to another object etc etc. Here's how it's initialized

{% highlight c %}
void kinit()
{
  initlock(&kmem.lock, "kmem");
  freerange(end, (void*)PHYSTOP);
}

void freerange(void *pa_start, void *pa_end)
{
  char *p;
  p = (char*)PGROUNDUP((uint64)pa_start);
  for(; p + PGSIZE <= (char*)pa_end; p += PGSIZE)
    kfree(p);
}

// Free the page of physical memory pointed at by pa,
// which normally should have been returned by a
// call to kalloc().  (The exception is when
// initializing the allocator; see kinit above.)
void kfree(void *pa)
{
  struct run *r;

  if(((uint64)pa % PGSIZE) != 0 || (char*)pa < end || (uint64)pa >= PHYSTOP)
    panic("kfree");

  // Fill with junk to catch dangling refs.
  memset(pa, 1, PGSIZE);

  r = (struct run*)pa;

  acquire(&kmem.lock);
  r->next = kmem.freelist;
  kmem.freelist = r;
  release(&kmem.lock);
}
{% endhighlight %}

and once initialised (by calling freerange in kinit with arguments being the first address after kernel and PHYSTOP), we end up with a list of pages that can be given out at request.
maybe a pic here ?????????

Now there's this function kalloc (it's not a system call). Kalloc acquires a lock (kmem.lock), grabs a page (if available) from kmem.freelist and returns an address of that page. At the same time it removes that page from freelist so it's no longer available for future kalloc calls. Also then the lock is released.

As for sbrk, it's a system call (calls growproc which in turn calls uvmalloc). Uvmalloc (already talked about this function in the post about memory mapping) calls kalloc (to allocate memory) and mappages (to then map that memory). So a program can request memory via sbrk call [^1]. That memory (given out by pages) comes from freelist. The problem is that there's just one freelist.
This can be problematic as for example if there's a bunch of memory hungry processes (them requesting memory via sbrk), it'll cause a lock contention (freelist is protected by a lock). The tests for this lab do exactly that (stress the memory allocator and cause a lock contention). The suggested solution to implement is a per cpu freelist.

How would that work ?
When kalloc is called (e.g. as a result of a call to sbrk) it grabs a page from a freelist that is assigned to the cpu it is currently running on. (e.g. if running on cpu nr 3 it uses a pagelist assign to cpu nr 3, same for other cpus) Except if the freelist for the given cpu is out of memory, in which case it'll try to 'steal' a page from one of the other freelists (assigned to other cpus). soooo, after the change. The new kinit looks like this: 

{% highlight c %}
// kmem is now an array as each cpu has its own freelist
struct {
  struct spinlock lock;
  struct run *freelist;
} kmem [NCPU];

void kinit()
{
  for(int i=0; i<NCPU; i++)
    initlock(&kmem[i].lock, "kmem");

  // kinit is executed just once (on startup)
  // below freerange calls kfree (in a loop) which adds pages to the freelist.
  // As it all happens on startup, initially all the pages will be
  // added to the same freelist (meaning initially they won't be distributed among freelists)
  freerange(end, (void*)PHYSTOP);
}
{% endhighlight %}


and kfree

{% highlight c %}
void kfree(void *pa)
{
  struct run *r;

  if(((uint64)pa % PGSIZE) != 0 || (char*)pa < end || (uint64)pa >= PHYSTOP)
    panic("kfree");

  // Fill with junk to catch dangling refs.
  memset(pa, 1, PGSIZE);

  r = (struct run*)pa;

  // only safe to call cpuid and use its result when interrupts are turned off.
  push_off();
  int id = cpuid();
  pop_off();

  acquire(&  kmem[id].lock);
  r->next = kmem[id].freelist;
  kmem[id].freelist = r;
  release(&  kmem[id].lock);
}
{% endhighlight %}

and also kalloc:
 
{% highlight c %}
void * kalloc(void)
{
  struct run *r;

  // only safe to call cpuid and use its result when interrupts are turned off.
  push_off();
  int id = cpuid();
  pop_off();

  acquire(&  kmem[id].lock);
  r = kmem[id].freelist;
  if(r)
    kmem[id].freelist = r->next;
  release(&  kmem[id].lock);

  if(!r) {
    for(int i=0; i<NCPU; i++){
      //if(i == id) continue;

      acquire(&kmem[i].lock);
      r = kmem[i].freelist;
      if(r)
        kmem[i].freelist = r->next;
      release(&kmem[i].lock);
      if(r) break;
    }
  }

  if(r)
    memset((char*)r, 5, PGSIZE); // fill with junk
  return (void*)r;
}
{% endhighlight %}

Spinlocks are a main protagonist in this post so here's a closer look:

{% highlight c %}
// Mutual exclusion lock.
struct spinlock {
  uint locked;       // Is the lock held?

  // For debugging:
  char *name;        // Name of lock.
  struct cpu *cpu;   // The cpu holding the lock.
};
{% endhighlight %}

and a function acquire:

{% highlight c %}
void acquire(struct spinlock *lk)
{
  push_off(); // disable interrupts to avoid deadlock.
  if(holding(lk)) panic("acquire");

  // On RISC-V, sync_lock_test_and_set turns into an atomic swap:
  //   a5 = 1
  //   s1 = &lk->locked
  //   amoswap.w.aq a5, a5, (s1)
  while(__sync_lock_test_and_set(&lk->locked, 1) != 0) ;

  // Tell the C compiler and the processor to not move loads or stores
  // past this point, to ensure that the critical section's memory
  // references happen strictly after the lock is acquired.
  // On RISC-V, this emits a fence instruction.
  __sync_synchronize();

  // Record info about lock acquisition for holding() and debugging.
  lk->cpu = mycpu();
}
{% endhighlight %}

There's also a function named release which (as the name suggests) releases the lock. Locks deserve a saparete post (e.g. spinlock vs sleeplock) so I'll hope to be able to write one.


[^1]: i guess most people are familiar with malloc which uses sbrk under the hood

