---
layout: post
title:  "print a page table"
date:   2024-12-01 19:39:55 +0100
categories: xv6
---

Memory mapping is a complicated subject. I plan to dedicate more posts to dive into it. Here I just show how to solve the xv6 labs task (I give some info on how it works under the hood but not much). A task is to print a page table.

A bit of theory first. Below to keep it short I'll be using the following conventions: pa - physical address, va - virtual address, pt - page table, pte - page table entry.

A need to translate from va to pa arises (for example) when CPU is to execute an instruction load (which has a va). How does this work ? The key thing is the register satp which points to pt (to be more precise we deduce a pa of pt from it). Pt is a page (continues chunk of memory that is 4096 bytes long). It has 512 entries (called pte-s). Each is 8 bytes long.
This is just a beginning of the story. Meaning the whole scheme has three levels of pt-s arranged in a tree-like manner. (there's a lot more stuff and details i do not want to go into here as i plan to do it in other posts) 

Here's how translation va to pa happens. We have (in satp) a pa of pt (let's call it pt1) and a va to translate. Do as follows 
1. Get 9 bits from va. They determine an index of pte to be grabbed (512 entries in pt so it makes sense as 2^9 = 512). Go ahead and grab that pte from pt1.
2. Translate that pte to a pa (below look at PTE2PA to see how) thus obtaining pa of another pt (let's call it pt2)!
3. Get another 9 bits from the va and repeat the above steps (use pt2 this time) to get pt3.
4. This step is similar to the previous one, except also 12 so called offset bits (which come from va) are used in combination with pte from pt3 to arrive at pa. Voila, now we've got pa! we've done va to pa translation!

Now on to the code I wrote. It resembles the function (already existing in xv6) freewalk.

```c
// below included for clarity
// kernel/riscv.h
// typeden uint64 *pagetable_t; // 512 PTEs
// #define PTE2PA(pte) (((pte) >> 10) << 12)

void printIndent(int depth, int idx)
{
  for (int i=0; i<=depth; i++) printf(" ..");
  printf("%d: ", idx);
}

// pte_t - uint64
// typedef uint64 *pagetable_t;
void vmprint(pagetable_t pagetable, unsigned int depth)
{
  if (!depth) printf("page table at pa %p \n", pagetable);

  // there are 2^9 = 512 PTEs in a page table.
  for(int i = 0; i < 512; i++){
    pte_t pte = pagetable[i];
    if((pte & PTE_V) && (pte & (PTE_R|PTE_W|PTE_X)) == 0){ // branch in the tree

      // this PTE points to a lower-level page table.
      uint64 child = PTE2PA(pte);

      printIndent(depth, i);
      printf("pte %p pa %p \n", pte, child);

      vmprint((pagetable_t)child, depth+1);
    }
    else if(pte & PTE_V)                                  // leaf in the tree
    {
      uint64 child = PTE2PA(pte);

      printIndent(depth, i);
      printf("pte %p pa %p \n", pte, child);
    }
  }
}
```

also so that pt of the process wih pid 1 is printed, the following line was added in the function exec just before return

```c
  if(p->pid==1) vmprint(p->pagetable, 0);
```


then start up xv6 and get the output

```
page table at pa 0x0000000087f6c000 
 ..0: pte 0x0000000021fda001 pa 0x0000000087f68000 
 .. ..0: pte 0x0000000021fd9c01 pa 0x0000000087f67000 
 .. .. ..0: pte 0x0000000021fda41b pa 0x0000000087f69000 
 .. .. ..1: pte 0x0000000021fd9817 pa 0x0000000087f66000 
 .. .. ..2: pte 0x0000000021fd9407 pa 0x0000000087f65000 
 .. .. ..3: pte 0x0000000021fd9017 pa 0x0000000087f64000 
 ..255: pte 0x0000000021fdac01 pa 0x0000000087f6b000 
 .. ..511: pte 0x0000000021fda801 pa 0x0000000087f6a000 
 .. .. ..510: pte 0x0000000021fdd007 pa 0x0000000087f74000 
 .. .. ..511: pte 0x0000000020001c0b pa 0x0000000080007000 
```

It's easy to see that only 39 bits (retrieve pte using 9 bits 3 times and 12 offset bits in the final step) from va are used for translation.
Above i'm talking about an user process but it applies to the kernel too. That's it for now, more posts about memory, mapping etc to follow.

