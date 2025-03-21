---
layout: post
title:  "add a system call"
date:   2025-03-02 19:39:55 +0100
categories: jekyll update
---

Some of the exercises concern system calls so I'm going to delve into this subject a bit.
The relevant code resides in usys.S (is created by a script usys.pl) and for example for a system call read looks like this 

{% highlight assembly %}
.global read
read:
 li a7, SYS_read
 ecall
 ret
{% endhighlight %}

It writes a value associated with this call (SYS_read) into a register a7 and executes an instruction ecall (which switches to the supervisor mode and jumps to uservec.
The procedure uservec happens to be mapped in same address in user and kernel (at this point we're in supervisor mode but still using user program's page table)

In uservec (trampoline.S) a bunch of stuff happens
- save process' regs in trapframe (do 'csrw sscratch, a0' to be able to use a0) 
- populate a couple of registers
- switch to kernel page table by setting satp accordingly
- jump to a function usertrap (which was first retrieved from the trapframe)

Then in usertrap (trap.c) check if we are here because of a system call. If so, call a procedure syscall (in syscall.c).

{% highlight c %}
void syscall(void)
{
  int num;
  struct proc *p = myproc();

  num = p->trapframe->a7;

  if(num > 0 && num < NELEM(syscalls) && syscalls[num]) {
    // lookup the system call function (syscalls holds pointers to all sys calls), call it, and store its return value in p->trapframe->a0
    p->trapframe->a0 = syscalls[num]();
  } else {
	// print sth
    p->trapframe->a0 = -1;
  }
}
{% endhighlight %}


It gets a value stored in trapframe that was initially passed in the register a7. For read that value will be SYS_read.
Then it calls an appropriate function. In case of read it's the following one (in sysfile.c)


{% highlight c %}
uint64 sys_read(void)
{
  struct file *f;
  int n;
  uint64 p;

  argaddr(1, &p);
  argint(2, &n);
  if(argfd(0, 0, &f) < 0)
    return -1;
  return fileread(f, p, n);
}
{% endhighlight %}


In the last line a call to fileread is made. But before that arguments are retrieved by calling argaddr, argint and argfd.
To see what's going on, let's consider an example call to read (e.g. in file.c) read(fd, addr, int).
Three arguments got passed and initially stored in registers. Then (in uservec) they were moved to trapframe (map regs to places in trapframe).
A call argaddr(1, &p) means - get the 2nd (indexing from 0, hence 1 passed) argument that is of address type and put it in a variable p.   
Once the work is finished, a return happens - usertrapret (trap.c) is called from usertrap.
It sets some registers and subsequently calls userret (trampoline.S) which switches back to the user page table, restores some registers and calls sret (return to the user mode)

There's a few registers that are of great importance (such as pc, sp, satp). Let's see where and when they are saved and restored.
pc - when a trap occurs (ecall), pc is saved in sepc. A function trap (trap.c) saves sepc in the trapframe and adds 4 bytes (since sepc points to the ecall instruction, and we want to return to the next instruction) Then in usertrapret (trap.c) sepc is overwritten with the value held in the trapframe. Finally when sret is executed, sepc value is put in pc.
sp - saved in the trapframe (offset 48) by uservec and later restored by userret 
satp - this value is constant for a process. when returning from the trap, usertrapret (trap.c) gets satp and passes it to userret (trampoline.S)


Now that I've shown how system calls work I'm going to focus on the task to solve. We are to add a new system call 'trace'.
An argument (named 'mask' and interpreted as bitmask) that is passed to this system call specifies which system calls to trace
(e.g. if bits 2 and 7 are set, every time a system call number 2 or 7 is invoked, a relevant information will be printed). 

A couple of files need to be changed e.g. a variable 'int mask' to be added to the struct proc. And the self-explanatory (i hope) code in sysproc.c


{% highlight c %}
uint64
sys_trace(void)
{
  int mask;
  argint(0, &mask);
  myproc()->mask = mask;
  return 0;
}
{% endhighlight %}


and in syscall.c


{% highlight c %}
void syscall(void)
{
  int num; // identifies system call
  struct proc *p = myproc();
  num = p->trapframe->a7;

  if(num > 0 && num < NELEM(syscalls) && syscalls[num]) {
    // Use num to lookup the system call function for num, call it, and store its return value in p->trapframe->a0
    p->trapframe->a0 = syscalls[num]();

    int mask = p->mask;
    char* names[] = { "dummy", "fork" ,"exit" ,"wait" ,"pipe" ,"read" ,"kill" ,"exec" ,"fstat" ,"chdir" ,"dup" ,"getpid" ,"sbrk" ,"sleep" ,"uptime" ,"open" ,"write" ,"mknod" ,"unlink" ,"link" ,"mkdir" ,"close" ,"trace" };

    // check if the system call is to be traced 
    if(mask & (1 << num)){
      printf("%d: syscall %s -> %d \n", p->pid, names[num], p->trapframe->a0);
    }

  } else {
	// print sth
    p->trapframe->a0 = -1;
  }
}
{% endhighlight %}


Hope it's clear (more or less).

