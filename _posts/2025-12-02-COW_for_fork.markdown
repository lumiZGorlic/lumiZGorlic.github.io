---
layout: post
title:  "COW for fork"
date:   2025-12-02 01:39:55 +0100
categories: jekyll update
---

La fonction fork (l'un des appels systeme) cree une copie (enfant) du processus courant. Pour faire cela la fonction uvmcopy est appelee. Elle a un role de copier les pages memoire de parent vers enfant. Tout ca c'est pas ideal (parce que toutes les pages sont dupliquees). Une meilleure idee consiste a faire partager les pages entre parent et enfant en modifiant les entrees de pagetable. (au milieu de creer de nouvelle pages immediatement). Dans cet exercise on va implementer cette approche (mais uniquement pour les pages marquees comme writeable 'W').

Voici la solution: Dans uvmcopy, les pages 'W' de parent sont remarquees 'RSW' (ainsi elles ne sont plus writeable). Les memes pages physiques sont mappees dans la pagetable de l'enfant (avec egalement le flag 'RSW'). Donc si on n'a pas (ni parent ni enfant) besoin d'ecrire dans ces pages ('RSW'), tout va bien: les pages restent partagees. Mais qu'est-ce qui se passe quand un processus (parent ou enfant) tente d'ecrire dans une page marquee 'RSW ?

L'ecriture provoque un page fault (c'est normal, car la page n'est plus 'W'). Pour gerer ca, on ajoute un cowhandler qui est invoque lors du 	page fault. Ce cowhandler cree une copie privee de la page, fait un update de l'entree de la pagetable (de processus qui tente d'ecrire) pour pointer vers cette nouvelle page, et marque la nouvelle page comme 'W'. Ensuite l'ecriture peut se faire.

A partir de la, le processus (qui a ecrit) a sa copie privee ('W') de la page (l'entree de la pagetable de l'autre processus n'est pas changee donc son mapping utilise la original page qui est 'RSW')
Au-desous les changement et explications:


kalloc.c - useReference (init, decrement)

{% highlight c %}
int useReference[PHYSTOP/PGSIZE];
struct spinlock ref_count_lock;

// Free the page of physical memory pointed at by pa,
// which normally should have been returned by a
// call to kalloc().  (The exception is when
// initializing the allocator; see kinit above.)
void kfree(void *pa)
{
  struct run *r;
  int temp;

  acquire(&ref_count_lock);
  useReference[(uint64)pa/PGSIZE] -= 1;
  temp = useReference[(uint64)pa/PGSIZE];
  release(&ref_count_lock);

  if(temp > 0) return;

  if(((uint64)pa % PGSIZE) != 0 || (char*)pa < end || (uint64)pa >= PHYSTOP)
    panic("kfree");

  // Fill with junk to catch dangling refs.
  memset(pa, 1, PGSIZE);

  r = (struct run*)pa;

  acquire(&kmem.lock);
  r->next = kmem.freelist;
  kmem.freelist = r;

  acquire(&ref_count_lock);
  useReference[(uint64)pa/PGSIZE] = 1;
  release(&ref_count_lock);

  release(&kmem.lock);
}
{% endhighlight c %}


trap.c - cowhandler

{% highlight c %}
int cowhandler(pagetable_t pagetable, uint64 va)
{
  char *mem;

  if(va >= MAXVA) return -1;
  pte_t* pte = walk(pagetable, va, 0);
  if (pte == 0) return -1;
  if( ((*pte & PTE_RSW) == 0) || ((*pte & PTE_U) == 0) || ((*pte & PTE_V) == 0)) return -1;
  if((mem = kalloc()) == 0) return -1;
  uint64 pa = PTE2PA(*pte);

  memmove((void*) mem, (const void*) pa, PGSIZE);

  kfree((void*) pa);

  uint flags = PTE_FLAGS(*pte);

  *pte = PA2PTE(mem) | flags | PTE_W;
  *pte &= ~PTE_RSW;

  // alternatively (to those 2 lines above) the below also works 
  //flags &= ~PTE_RSW;
  //flags |= PTE_W;
  //uvmunmap(pagetable, PGROUNDDOWN(va), 1, 0);
  //mappages(pagetable, PGROUNDDOWN(va), PGSIZE, (uint64)mem, flags);

  return 0;
}

void usertrap(void)
{
  int which_dev = 0;

  if((r_sstatus() & SSTATUS_SPP) != 0)
    panic("usertrap: not from user mode");

  // send interrupts and exceptions to kerneltrap(),
  // since we're now in the kernel.
  w_stvec((uint64)kernelvec);

  struct proc *p = myproc();
  
  // save user program counter.
  p->trapframe->epc = r_sepc();
  
  if(r_scause() == 8){
    // system call

    if(killed(p))
      exit(-1);

    // sepc points to the ecall instruction,
    // but we want to return to the next instruction.
    p->trapframe->epc += 4;

    // an interrupt will change sepc, scause, and sstatus,
    // so enable only now that we're done with those registers.
    intr_on();

    syscall();
  } else if(r_scause() == 15){
    uint64 va = r_stval();
    if(va >= p->sz) p->killed = 1;
    int ret = cowhandler(p->pagetable, va);
    if(ret == -1) p->killed = 1;

  } else if((which_dev = devintr()) != 0){
    // ok
  } else {
    printf("usertrap(): unexpected scause %p pid=%d\n", r_scause(), p->pid);
    printf("            sepc=%p stval=%p\n", r_sepc(), r_stval());
    setkilled(p);
  }

  if(killed(p))
    exit(-1);

  // give up the CPU if this is a timer interrupt.
  if(which_dev == 2)
    yield();

  usertrapret();
}

{% endhighlight c %}


vm.c - uvmcopy creer PTE_RSW quand fork 

{% highlight c %}
int uvmcopy(pagetable_t old, pagetable_t new, uint64 sz)
{
  pte_t *pte;
  uint64 pa, i;
  uint flags;
////////////////////////////////////////////////////////////////////////////////////////////////////
  //char *mem; // enlevee, pas besoin, on ne va pas faire kalloc 

  for(i = 0; i < sz; i += PGSIZE){
    if((pte = walk(old, i, 0)) == 0)
      panic("uvmcopy: pte should exist");
    if((*pte & PTE_V) == 0)
      panic("uvmcopy: page not present");

////////////////////////////////////////////////////////////////////////////////////////////////////
    // si c'est W, efface W, marque RSW 
    if (*pte & PTE_W) {
      *pte &= ~PTE_W;
      *pte |= PTE_RSW;
    }

    pa = PTE2PA(*pte);
    flags = PTE_FLAGS(*pte);

////////////////////////////////////////////////////////////////////////////////////////////////////
    // pas de kalloc, moving, mapping etc
    //if((mem = kalloc()) == 0)
    //  goto err;
    //memmove(mem, (char*)pa, PGSIZE);
    //if(mappages(new, i, PGSIZE, (uint64)mem, flags) != 0){
    //  kfree(mem);
    //  goto err; }

////////////////////////////////////////////////////////////////////////////////////////////////////
    // il faut faire book keeping / keep track
    acquire(&ref_count_lock);
    useReference[(uint64)pa/PGSIZE] += 1;
    release(&ref_count_lock);

////////////////////////////////////////////////////////////////////////////////////////////////////
    // mapping pour nouveau ptable
    if(mappages(new, i, PGSIZE, (uint64)pa, flags) != 0){
      goto err;
    }

  }
  return 0;

 err:
  uvmunmap(new, 0, i / PGSIZE, 1);
  return -
}

{% endhighlight c %}


vm.c - copyout faire ce que on fait dans cowhandler 

{% highlight c %}
// Copy from kernel to user.
// Copy len bytes from src to virtual address dstva in a given page table.
// Return 0 on success, -1 on error.
int
copyout(pagetable_t pagetable, uint64 dstva, char *src, uint64 len)
{
  uint64 n, va0, pa0;
  pte_t *pte;

  while(len > 0){
    va0 = PGROUNDDOWN(dstva);
    if(va0 >= MAXVA)
      return -1;
    pte = walk(pagetable, va0, 0);
    if(pte == 0 || (*pte & PTE_V) == 0 || (*pte & PTE_U) == 0 ||
       (*pte & (PTE_W | PTE_RSW)) == 0)
      return -1;
    pa0 = PTE2PA(*pte);

    struct proc *p = myproc();
    if(checkcowpage(va0, pte, p))
    {
      char* mem;
      if((mem = kalloc()) == 0)
      {
        p->killed = 1;
        return -1;
      }
      else
      {
        memmove(mem, (char*)pa0, PGSIZE);

        uint flags = PTE_FLAGS(*pte);
        uvmunmap(pagetable, va0, 1, 1);

        *pte  = PA2PTE(mem) | flags | PTE_W;
        *pte &=  ~PTE_RSW;

// below an alternative to the 4 lines above
//        kfree((void*) pa0);
//        uint flags = PTE_FLAGS(*pte);
//        *pte = PA2PTE(mem) | flags | PTE_W;
//        *pte &= ~PTE_RSW;


        pa0 = (uint64)mem;
      }
    }

    n = PGSIZE - (dstva - va0);
    if(n > len)
      n = len;
    memmove((void *)(pa0 + (dstva - va0)), src, n);

    len -= n;
    src += n;
    dstva = va0 + PGSIZE;
  }
  return 0;
}

{% endhighlight c %}

