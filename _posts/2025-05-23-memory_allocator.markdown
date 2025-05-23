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
void
freerange(void *pa_start, void *pa_end)
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
void
kfree(void *pa)
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

