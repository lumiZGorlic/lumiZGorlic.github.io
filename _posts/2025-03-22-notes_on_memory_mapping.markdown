---
layout: post
title:  "notes on memory mapping"
date:   2025-03-22 01:39:55 +0100
categories: jekyll update
---

Here's a post (related to the previous one about page table) about memory, mapping etc. It'll be a collection of notes which i hope will be consistent.

First of all why is a tree structure used for the memory mapping ? Why not use a linear data structure for that ? Let's consider a 32 bits address space. A process' address space size would be over 4 GB (2^32 = 4294967296 bytes). Now a page size is 4 Kb (4096 bytes which is 2^12). How many pages will this space be divided into ? Let's divide - 4294967296 / 4096 = 1048576. Each page requires a page table entry which takes up 4 bytes. So that linear data structure would have a size 1048576 * 4 = 4 194 304 bytes. This is over 4 MB. That's a lot. If we realise that very often processes use just a small part of the available address space and there's lots of addresses that do not need to be mapped (so they don't need page table entries). Hence the choice of a tree structure seems justified (a lot of space saved this way).

Xv6 runs on RISC-V in a Sv39 mode. That means that only the bottom 39 bits of a 64-bit virtual address are used. In a nutshell the translation (va -> pa) goes like this - use first 9 (A pagetable has exactly 512 entries. 2^9 = 512 ) bits in va as an index into the 1st level page table (pointed to by satp). Retrieve a page table entry and deduce a physicall address (pointing to 2nd level page table) from it. Do same thing at 2nd level and then at 3rd. Thus we end up with a physical address of a page. Then the remaining 12 bits determine a specific byte within that page. Should make sense. 9 (index into 1st level page table) + 9 (2nd level) + 9 (3rd level) + 12 (the byte) = 39 (4096 = 2^12)

A question that occurred to me is - can mapping have 1 or 2 levels instead of normal 3 ? The answer is yes. In some cases (e.g. small amount of memory to b mapped) it might be more practical to use fewer levels.

A piece of hardware called memory managment unit (in short mmu) is behind all this translation happening pretty much all the time (interesting to note that program counter also holds a virtual address. Before a fetch is performed that address needs to be translated to a physical one).

When the function exe is called, it'll start up a process. But first it needs to load stuff into memory and set up the mapping. Let's see how it's done. Here's the functions in question (some code is left out). Note the use of proc_pagetable and uvmalloc. I could add a picture of process memory here, it would be easy to see the calls below set that mapping up. Perhaps something todo.

{% highlight c %}
int exec(char *path, char **argv)
{
  // do stuff

  pagetable_t pagetable = 0, oldpagetable;

  if((pagetable = proc_pagetable(p)) == 0) goto bad;

  // Load program into memory.
  for(i=0, off=elf.phoff; i<elf.phnum; i++, off+=sizeof(ph)){
    // do stuff

	// allocate memory for the next segment of the file to be loaded
    if((sz1 = uvmalloc(pagetable, sz, ph.vaddr + ph.memsz, flags2perm(ph.flags))) == 0)
      goto bad;
	// load segment
  }

  // Allocate two pages at the next page boundary.
  // Make the first inaccessible as a stack guard.
  // Use the second as the user stack.
  sz = PGROUNDUP(sz);
  uint64 sz1;
  if((sz1 = uvmalloc(pagetable, sz, sz + 2*PGSIZE, PTE_W)) == 0) goto bad;
  sz = sz1;

  uvmclear(pagetable, sz-2*PGSIZE);
  sp = sz;
  stackbase = sp - PGSIZE;

  // Commit to the user image.
  oldpagetable = p->pagetable;
  p->pagetable = pagetable;

 bad:
   // do stuff
}
{% endhighlight %}

They look like this. Here we see that in uvmalloc memory is allocated by kalloc and then mappages is called. In proc_pagetable the pages to be mapped by mappages already exist (trampoline and trapframe) so no need to call kalloc.

{% highlight c %}
// Create a user page table for a given process, with no user memory,
// but with trampoline and trapframe pages.
pagetable_t proc_pagetable(struct proc *p)
{
  pagetable_t pagetable;

  // An empty page table.
  pagetable = uvmcreate();
  if(pagetable == 0) return 0;

  // map the trampoline code (for system call return) at the highest user virtual address
  if(mappages(pagetable, TRAMPOLINE, PGSIZE, (uint64)trampoline, PTE_R | PTE_X) < 0)
    // call uvmfree and return 0

  // map the trapframe page just below the trampoline page, for trampoline.S.
  if(mappages(pagetable, TRAPFRAME, PGSIZE, (uint64)(p->trapframe), PTE_R | PTE_W) < 0)
    // call uvmunmap, uvmfree and return 0

  return pagetable;
}

// Allocate PTEs and physical memory to grow process from oldsz to
// newsz, which need not be page aligned.  Returns new size or 0 on error.
uint64
uvmalloc(pagetable_t pagetable, uint64 oldsz, uint64 newsz, int xperm)
{
  char *mem;
  uint64 a;

  if(newsz < oldsz) return oldsz;

  oldsz = PGROUNDUP(oldsz);
  for(a = oldsz; a < newsz; a += PGSIZE){
    mem = kalloc();
	// if fail, dealloc and retur 0
    memset(mem, 0, PGSIZE);

    if(mappages(pagetable, a, PGSIZE, (uint64)mem, PTE_R|PTE_U|xperm) != 0)
	  // free, dealloc and return 0
  }

  return newsz;
}
{% endhighlight %}

Mappages does mapping (can map more than 1 page) by
- calling walk (which is called with 'alloc' set to 1, so kalloc will be used to allocate new page tables if they don't exist) which returns the address of pte that corresponds to virtual address va 
- store the physicall address in that pte
A flag PTE_V indicates whether the PTE is present, hence it's set when pte is assigned an address of newly allocated pagetable or a physicall address to be mapped (passed to mappages). 

{% highlight c %}
// Create PTEs for virtual addresses starting at va that refer to
// physical addresses starting at pa. va and size might not
// be page-aligned. Returns 0 on success, -1 if walk() couldn't
// allocate a needed page-table page.
int mappages(pagetable_t pagetable, uint64 va, uint64 size, uint64 pa, int perm)
{
  uint64 a, last;
  pte_t *pte;

  a = PGROUNDDOWN(va);
  last = PGROUNDDOWN(va + size - 1);

  for(;;){
    if((pte = walk(pagetable, a, 1)) == 0) return -1;
    if(*pte & PTE_V)                       panic("mappages: remap");

    *pte = PA2PTE(pa) | perm | PTE_V; // store the physical address in pte

    if(a == last) break;
    a += PGSIZE;
    pa += PGSIZE;
  }
  return 0;
}
{% endhighlight %}

the function walk is similar to what i wrote in the post 'print a page table' (iterative vs recursive ????).

{% highlight c %}
// The risc-v Sv39 scheme has three levels of page-table
// pages. A page-table page contains 512 64-bit PTEs.
// A 64-bit virtual address is split into five fields:
//   39..63 -- must be zero.
//   30..38 -- 9 bits of level-2 index.
//   21..29 -- 9 bits of level-1 index.
//   12..20 -- 9 bits of level-0 index.
//    0..11 -- 12 bits of byte offset within the page.
pte_t * walk(pagetable_t pagetable, uint64 va, int alloc)
{
  if(va >= MAXVA) panic("walk");

  for(int level = 2; level > 0; level--) {
    pte_t *pte = &pagetable[PX(level, va)];             // get pte (PX extracts an index into pagetable from va at the given level)
    if(*pte & PTE_V) {
      pagetable = (pagetable_t)PTE2PA(*pte);            // get pa for the pagetable at the next level
    } else {
      if(!alloc || (pagetable = (pde_t*)kalloc()) == 0) // if alloc set then allocate a new pagetable
        return 0;
      memset(pagetable, 0, PGSIZE);
      *pte = PA2PTE(pagetable) | PTE_V;                 // put the newly created pagetable's pa into pte
    }
  }
  return &pagetable[PX(0, va)];
}
{% endhighlight %}

Hope the above is (more or less) clear. A few sites that I found useful
?????
https://marz.utk.edu/my-courses/cosc130/lectures/virtual-memory/
https://zolutal.github.io/understanding-paging/

