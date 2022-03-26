---
title: "hardware memory models for programmers"
date: 2022-03-20T20:43:25Z
draft: false
---

[originally posted 2013](https://isaiahperumalla.wordpress.com/2013/06/24/hardware-memory-models-for-programmers/) 

Nearly all computer systems now a days have hardware which have multi core chips and shared memory. in this post I hope to summarize things i learnt from a [x86-TSO paper](https://www.cl.cam.ac.uk/~pes20/weakmemory/cacm.pdf), and hope to provide a high level overview of TSO (total store order) model which is most common on x86 processors. What i really like about this paper is it provices a simple abstract model software developers can use to reason about low-level concurrency code running on x86 modern multi-core shared memory systems.

Most programmer are aware of modern multi-core systems have hierarcial memory subsystem and level of cpu caches and numerous performance optimizations that have observable consequence for multi-threaded concurrent programs. 
From a correctness point of  view most of the internal components are invisible or cannot be observed by single threaded software , one of the components to keep a note of is the store buffers of each core. 
Even with the hierarchy of fast caches to main memory , the core often stall while accessing memory, to further hide this latency each core has it own private load and store buffer which basically a FIFO queue which buffer pending writes and reads from memory. We will see later that store buffer does have observable consequences for multi-threaded programs.

## why memory model

Memory model allow programmers to reason about the correctness of their programs, it also help programmer get most out the performance optimizations modern multi-core systems  can make. The x86 processor have most strongest guarentees and most intutive but even with this there are some non-intuitve results which are legal. Lets now explore the allowed globally visible states from program below running on a x86 intel processor (i tried this out on an intel core i3)

// x and y are initialised to 0

|CORE-0	| CORE-1|
|-------|-------|
|x = 1	| y = 1 |
|r0 = y	| r1 = x|

In code it would look something like this
```
int x, y = 0;
int r0, r1;
 
void thread0 ()
{
  x = 1;
  r0 = y;
  
}

void thread1 ()
{
  y = 1;
  r1 = x;
  
}

```

The interesting thing in the example above is there is **only one writer** to any given memory location at any time. On a multi-core processor what could the end results of x and y be?  Intuitively we can consider all possible in program-order executions,   see that any of the values below could be expected.

1. Thread0 runs first all way through r0=0, r1=1 
2. Thread1 runs first all way through r0=1, r1=0
3. any other interleaving opertion can only result in r0=1 , r1=1 
Interleaving possiblites 

|STEP-1	        | STEP-2        | STEP-3        |	STEP-4	 | (R0, R1)|
|---------------|---------------|---------------|------------|---------|
|core-1: y=1	  |core-0: x=1	|core-1: r1=x	|core-0: r0=y|	(1,1)  |
|core-1: y=1	  |core-0: x=1	|core-0: r1=y	|core-1: r0=x|	(1,1)  |
|core-0: x=1	  |core-1: y=1	|core-0: r0=y	|core-1: r1=x|	(1,1)  |
|core-0: x=1	  |core-1: y=1	|core-1: r1=x	|core-0: r0=y|	(1,1)  |


The sequential semantics that is intuitive for most us is called sequential consistency model.

As we can see it the table above , assuming each processor executed in-program order , There is no interleaving execution that could produce a state where (r0,r1) = (0,0). If the state were to occur (r0,r1) = (0,0) , it would appear from a software point of view as if either thread-0 or thread-1 effectively executed their instructions out of sequence. 
1. compiler reorders the write and read operation 

we can stop compiler from doing any reordering , we can verify this by examing the assembly ouput from gcc compiler

[gist](https://gist.github.com/isaiah-perumalla/5809246)

```c {linenos=table,hl_lines=[10 18],linenostart=1}
#include <pthread.h>
#include <stdio.h>
 
int x, y = 0;
int r0, r1;
 
void *core0 (void *arg)
{
  x = 1;
  asm volatile ("" ::: "memory"); // ensure GCC compiler will not reorder 
  r1 = y;
  return 0;
}
 
void *core1 (void *arg)
{
  y = 1;
  asm volatile ("" ::: "memory"); // ensure GCC compiler will not reorder 
  r0 = x;
  return 0;
}
 

int main (void)
{
  pthread_t thread0, thread1;
  while (1) {
    x = y = 0;
    //Start threads
    pthread_create (&thread0, NULL, core0, NULL);
    pthread_create (&thread1, NULL, core1, NULL);

    //wait for threads to complete
    pthread_join (thread0, NULL);
    pthread_join (thread1, NULL);
 
    if (r0 == 0 && r1 ==0) {
      printf ("(r0=%d, r1=%d)\n", r0, r1);
      break;
    }

  }
  return 0;
}

```


Since we are focusing on the behavior of the hardware, we want to ensure the gcc compiler does not reorder any of the instruction,

 `asm volatile ("" ::: "memory");`

directive on lines 10 and 18 ensure compiler optimization will not reorder the statements.

Compiling the above code with gcc with 02 optimizations
``` shell 
gcc -02 x86mem.c -o x86mem -pthread
```


looking at the assembly output of the code its clear write followed by read instructions are *not reorder* 

[https://gcc.godbolt.org/z/W1bh3rE3a](https://gcc.godbolt.org/z/W1bh3rE3a)
``` asm {linenos=table,hl_lines=["5-6" "15-16"],linenostart=1}
core0:
        push    rbp
        mov     rbp, rsp
        mov     QWORD PTR [rbp-8], rdi
        mov     DWORD PTR x[rip], 1   ## write value 1 to x
        mov     eax, DWORD PTR y[rip] ## read contents of y into  register
        mov     DWORD PTR r1[rip], eax
        mov     eax, 0
        pop     rbp
        ret
core1:
        push    rbp
        mov     rbp, rsp
        mov     QWORD PTR [rbp-8], rdi
        mov     DWORD PTR y[rip], 1 ## write value 1 to y
        mov     eax, DWORD PTR x[rip] ## read contents of x into  register
        mov     DWORD PTR r0[rip], eax
        mov     eax, 0
        pop     rbp
        ret

```
## Dont forget the Store buffer !

What we seeing about are visible effects of store buffer. Store buffer is just a private FIFO queue to schedule write to memory subystem, this is not a side effect of cpu cache coherency but it is the effects of private store buffers of each core. 
as we demonstrated in the example the write to memory by each core are queued its private store buffer, while each core can read its most recent buffered write ,  the writes in the buffer are invisible to other cores. So before each core’s buffered writes have propagated to the memory subsystem, each core could already executed the next read instruction,  this effectively has the same effect as a memory read instruction getting ahead of an older memory write instruction, according to the x86 TSO memory model this is valid execution. So it clear from this example a programmer model of the hardware must account for the store buffer. To understand and reason about the x86 memory model the following simpler hardware model is sufficient for a programmer.

 
![programmers view](/imgs/memory/memorymodel.png)

In the abstract model above the programmer can simply view each core (hardware thread in this context) as having it own private store FIFO buffers, all writes get queued into the store buffer and each core can ‘snoop its own buffer’ , that is read its most recent buffered write directly from its buffer (other cores will not see the buffered writes until it is propagated to memory). This is why running the program above on a single cpu core will never result in state `r0=0, r1=0`

 As the example demonstrated the x86 memory model in a mulit-core execution the store buffers are made visible to the programmer, the consequence of this is the x86 model, has slightly weaker memory model than sequential consistency called the Total store order. In a TSO model following operations must be visible to all cores as if they were executed sequentially

* Load  are not reordered with other loads
* Store  are not reordered with other stores as every write is queued in a FIFO buffer
* Loads can be reordered with earlier stores to different memory location. Load and stores to same memory location will not be reordered
* Memory ordering obeys causality (ie stores are seen in consistent order by other processors)

## What can programmers do  ?

Since a Store followed by a Load can appear out of order to other cores, however programmers may wish to ensure load and store instructions to be ordered across all cores , the x86 processor have a ‘mfence’ (memory fence) instruction. the mfence instruction will ensure a core will not only reorder the instructions, but also will ensure that any *buffered stores in the cores private store buffer are propagated to the memory subsystem* before executing instructions after mfence. with this instruction we can strengthen the memory consistency model when required by the software. By adding the mfence instruction to our previous example , it should never be possible to get a state where (r1,r2) = (0,0)

``` C {linenos=table,hl_lines=[11 19],linenostart=1}

#include <pthread.h>
#include <stdio.h>
 
int x, y = 0;
int r0, r1;
 
void *core0 (void *arg)
{
  x = 1;
  asm volatile ("mfence" ::: "memory"); // ensure compiler or hardware will not reorder, store buffer is drained
  r1 = y;
  return 0;
}
 
void *core1 (void *arg)
{
  y = 1;
  asm volatile ("mfence" ::: "memory"); // ensure compiler or hardware will not reorder, store buffer is drained
  r0 = x;
  return 0;
}
 
 
int main (void)
{
  pthread_t thread0, thread1;
  while (1) {
    x = y = 0;
    //Start threads
    pthread_create (&thread0, NULL, core0, NULL);
    pthread_create (&thread1, NULL, core1, NULL);
 
    //wait for threads to complete
    pthread_join (thread0, NULL);
    pthread_join (thread1, NULL);
 
    if (r0 == 0 && r1 ==0) {
      printf ("(r0=%d, r1=%d)\n", r0, r1);
      break;
    }
 
  }
  return 0;
}

```

The above program should not terminate, as there can never be a case where (r1,r2) = (0,0).

x86 processor also have a number of atomic instructions, such as XCHG which swaps the data between two memory locations atomically at the hardware . All atomic operations implicitly also drain the store buffer and prevents core from reordering instructions. In the next post we will go through practical examples of using memory models to reason about correctness and improve performance.

