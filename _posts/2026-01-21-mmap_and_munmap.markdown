---
layout: post
title:  "mmap and munmap"
date:   2026-01-21 01:39:55 +0100
categories: jekyll update
---


Now we are to implement mmap (and munmap) system calls - actually just a subset of their features related to mapping files. A fully fledged mmap is very powerful and has many use cases, for example it allows us to: 
1) memory map files (thanks to that, we can treat a file as contiguous memory region and thus avoid calls to read and write)
2) share memory between processes (used when processes need fast access to huge data sets e.g. in case of databases)
3) allocate memory (as an alternative to malloc)
4) map code and data segments (which is for example what kernel does when an user process uses shared libraries)
and i guess many many more.

Here's mmap signature 

void *mmap(void *addr, size_t len, int prot, int flags,
           int fd, off_t offset);

The lab assignment explains the params and their possible values:
addr   - in our lab, the only valid value is 0 (meaning the kernel will decide at which virtual address to map the file) 
len    - the number of bytes to map. might differ from the file's length
prot   - whether the memory should be mapped readable, writeable, and/or executable
flags  - in our case 2 possible values MAP_SHARED (write changes to the mapped memory back to the file) or MAP_PRIVATE (do not write changes to the file)
fd     - open file descriptor of the file to map
offset - always 0. (starting point in the file at which to map)
mmap returns address at which the file was mappped (if success), or 0xffffffffffffffff (if fail).

Now munmap

int munmap(void *addr, size_t len);

Munmap removes mmap mappings in the specified address range. If the the memory has been modified and the mapping is MAP_SHARED, the modifications get written back to the file. We can assume that munmap is called either to unmap at the start, or at the end, or the whole region (but not punch a hole in the middle of a region). 

OK, so what do we do to make it work ???? Let's get a few important points down.

First of all, the usual boilerplate stuff that we normally add when implementing a new system call needs to be added (covered in post 'add a system call').

Next, we need to be able to keep track of what mmap has mapped. For that we define a structure (called vma) that holds all the relevant details of the mapping. Each process then has an array of these VMAs, with a fixed maximum number of entries.

Now let's look at mmap implementation. When called, mmap first finds an unused region in the process's address space. Then finds a free vma struct (in the array) and fills it with details about the new mapping. It does not actually do the job we might expect (calling kalloc to get memory, mapping it in the page table, and finally copying the file contents). So where does all of that happen ? The answer is the allocation and copying do not happen until at some point the memory is actually needed (the process tries to access it). At that point, a page fault occurs (attempt to access unmapped memory) and the control enters page fault handling code which is where the memory allocation and file copy finally happen! So we have a lazy initialization technique at play here (like in the post 'COW for fork').

Let's see the code now


vma struct is added in proc.h and its initialization (at a process start up) is in proc.c

{% highlight c %}
uint64
sys_mmap(void)
{
  uint64 addr;
  int len, prot, flags, offset;
  struct file *f;

  argaddr(0, &addr);
  argint(1, &len);
  argint(2, &prot);
  argint(3, &flags);
  if(argfd(4, 0, &f) < 0) return -1;
  argint(5, &offset);

  struct proc* p = myproc();

  // Check whether this operation is legal
  if((flags==MAP_SHARED && f->writable==0 && (prot & PROT_WRITE))) return -1;

  // Find an empty VMA struct.
  int idx = 0;
  for(;idx<VMA_MAX;idx++)
    if(p->vma_array[idx].valid==0)
      break;
  if(idx==VMA_MAX)
    panic("All VMA struct is full!");
  // printf("sys_mmap: Find available VMA struct, idx = %d\n", idx);

  // Fill this VMA struct.
  struct vma* vp = &p->vma_array[idx];
  vp->valid = 1;
  vp->len = len;
  vp->flags = flags;
  vp->off = offset;
  vp->prot = prot;
  vp->f = f;
  filedup(f);

  p->vma_top_addr-=len;
  vp->addr = p->vma_top_addr; // The usable user virtual address.
  // printf("sys_mmap: Successfully mapped a file, with addr=%p, len=%x\n", vp->addr, vp->len);

  return vp->addr;
}
{% endhighlight c %}


in sysfile.c resides implementation of mmap and munmap. hope with the comments it's all clear.

{% highlight c %}
uint64
sys_munmap(void)
{
  uint64 addr;
  int len;
  argaddr(0, &addr);
  argint(1, &len);


  struct proc* p = myproc();

  struct vma* vp = 0;
  // Find the VMA struct that this file belongs to.
  for(struct vma *now = p->vma_array; now < p->vma_array + VMA_MAX; now++)
  {
    // printf("usertrap: VMA, addr=%p, len=%x, valid=%d\n", now->addr, now->len, now->valid);
    if(now->addr <= addr && addr < now->addr + now->len
        && now->valid)
    {
      vp = now;
      break;
    }
  }

  if(vp)
  {
    if( walkaddr( p->pagetable , addr ) != 0)
    {
      // Write back and unmap.
      if(vp->flags==MAP_SHARED) filewrite(vp->f, addr, len);
      uvmunmap(p->pagetable, addr, len/PGSIZE, 1);
      return 0;
    }


    // Update the file's refcnt.
    vp->refcnt -= 1;
    if(vp->refcnt) // set the vma struct to invalid.
    {
      fileclose(vp->f);
      vp->valid = 0;
    }
    return 0;
  }
  else
  {
    panic("Cannot find a vma struct representing this file!");
  }
}
{% endhighlight c %}


and as exaplined above, in case of mmap the allocation and copying of the file is postponed till process needs to access that memory. That's why it's in page fault handling code.

{% highlight c %}
  } else if(r_scause()==13){
    // Handle load page fault (when a file is mapped in the vitrual address space, but the physical page is not loaded).
    uint64 target_va = r_stval();
    struct vma* vp = 0;
    // printf("usertrap: Trying to visit va %p\n", target_va);

    // Find the VMA struct that this file belongs to.
    for(struct vma *now = p->vma_array; now < p->vma_array + VMA_MAX; now++) {
      // printf("usertrap: VMA, addr=%p, len=%x, valid=%d\n", now->addr, now->len, now->valid);
      if(now->addr <= target_va && target_va < now->addr + now->len
         && now->valid) {
        vp = now;
        break;
      }
    }

    if(vp) {
      // Allocate a page into physical memory, and map it to the virtual memory.
      uint64 mem = (uint64)kalloc();
      memset((void *)mem, 0, PGSIZE); // Set the page to all zero.
      if(mappages(p->pagetable, target_va, PGSIZE, mem, PTE_U|PTE_V|(vp->prot<<1))<0)
        panic("Cannot map a virtual page for the file!");

      // Load the content of the page.
      vp->refcnt += 1;
      ilock(vp->f->ip);
      readi(vp->f->ip, 0, mem, target_va-vp->addr, PGSIZE); // Load a file page from the disk
      iunlock(vp->f->ip);
    }
    else {
      printf("Unable to find the VMA struct that the file belongs to!\n");
      goto unexpected_scause;
    }
  } else if((which_dev = devintr()) != 0){
    // ok

  } else {
unexpected_scause:
    printf("usertrap(): unexpected scause %p pid=%d\n", r_scause(), p->pid);
    printf("            sepc=%p stval=%p\n", r_sepc(), r_stval());
    setkilled(p);
  }
{% endhighlight c %}


That's it for now!

