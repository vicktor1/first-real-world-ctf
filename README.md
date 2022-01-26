/************************************************************
    Real world CTF 2022: simple virtual machine
************************************************************/

this was my First real world CTF :) and I not solved completely
challenge during the CTF time but I solved 80% uptill changing
variable values and writing memory but Afterwards I completed
whole as I learn so much from this challenge. 

Firstly, I don't understand where to start because TIFC
So, I heard of fuzzing and also wrote some simple fuzzing C program
so I started with that I wrote simple fuzzing with some crafted random 
input in program as program.(fuzz.c)
 
As the program is crashing many times with segfault so I
got very excited and started examining testcases and run
that test case in gdb, As we are already given with source code.

While executing program in gdb I got its first glance what it is
doing with the input and then I was getting some understanding
of the source code I started reading whole program and 

I seen some thing weird about load instruction where It taking
input from the user without for where to read and where
to write in memory but it is checking anything about it.

But, still It is not as easy as it seems in first glance when
I see that It is using (int32_t) as input so we can't
write anywhere in the memory directly we need some kind of 
approach.

So first thing, first We need to write some program in order
to make our pain of converting instruction into virtual machine
code. So written another C program that will convert our
instruction into compiler bytecode. (compiler.c)


binary ELF

                               uninitialize 
                              global variable
                                    |                                                           some
     binary       read only         | preinitialize                                            kernel
      code          data            | global variable                 some loaded library       shit
        |             |             |       |                                 |                   |
        v             v             v       v                                 v                   v
  ---------------------------------------------------------------------------------------------------
  |              |              |        |        |----------|           |       |            |     |
  |              |              |        |        |----------|           |       |            |     |
  |  .text       |   .rodata    |  .data | .bss   |----------|  heap     |       |  stack     |     |
  |              |              |        |        |----------|           |       |            |     |
  |              |              |        |        |----------|           |       |            |     |
  |              |              |        |        |----------|           |       |            |     |
  ---------------------------------------------------------------------------------------------------
 0x00             Increasing addresses ----->                                          ^     ^    0x16f
                                                                                       |     |
                                                                                     $rsp   $rbp

                                                                                  <-stack 
                                                                                   pointer

                                                                                  ( means stack
                                                                                  grows toward
                                                                                 horizontally left )

Now we have everything getting started with challenge.
our compiler and program that main.c which you can compile.

In vm.c let's look at source code of virtual machine
mainly in instruction load, store, gload and gstore


load case:
----------
    offset = vm->code[ip++];
    vm->stack[++sp] = vm->call_stack[callsp].locals[offset];


    In above to lines It takes input from user as bytecode
    and the execute this cases.
    where,
    offset is user inputted argv 1 i.e (load argv1)

    2nd line that will store data from 
     vm->call_stack[callsp].locals[offset];
    and offset is already controlled by us.
    so we can read from anywhere uptill 2 ^ 32


gload case:
----------
    addr = vm->code[ip++];
    vm->stack[++sp] = vm->globals[addr];
 
   similiar as load but we are using vm->globals to
    read data where we are not checking address
    variable which is from user input

store case:
----------
    offset = vm->code[ip++];
    vm->call_stack[callsp].locals[offset] = vm->stack[sp--];
    
    In this case we are taking offset as second argv of 
    (store argv2) bytecode.
    
    now we are overwriting data at some offset by 
    value from our virtual machine stack.


gstore case: 
-----------
    addr = vm->code[ip++];
    vm->globals[addr] = vm->stack[sp--];

    Similiar as store



Exploitation approach:
---------------------
We can overwrite up to range integer of 32 bit i.e. 4byte,
and Program has alsr enabled, stack canary enable, Non-executable stack,
Non-writable code section.

But above things will not affect as at all we are using 
array indexes means our code crafted data will be loaded
at our deceded position.

So our approach is by using return oriented programming which
works like this:


    0x45    call foo
--> 0x50    some_instruction
|
|
next rip

      $rsp->| 0x50  |
            |-------|
            |||||||||
            ---------

    We have call instruction that stores
    address of next instruction  on stack 
    before jumping to called function.


    0x00    ret
    ret is equivalent to pop rip

        | 0x50  | jmp to 0x50
        |-------|
  $rsp->|||||||||   
        ---------
    vice versa ret instruction that pops
    instruction from the stack and jump
    to that offset.

We have two read-write from location:
    vm->call_stack[callsp].locals[offset];
    vm->globals[addr];

In order to read-write to some stack offset we 
need some relative offset to do this we
have vm->code that is going to help us because
it stores stack offset in it.

vm->globals is perfectly suitable member for
this task which is pointer so it stores address
and we can overwrite that address by address of
vm->code.

In gdb we can get offset of all the three

(gdb) print &vm->call_stack[callsp].locals[0]
$9 = (int *) 0x555befd72234
(gdb) print &vm->globals
$10 = (int **) 0x555befd712b0
(gdb) print &vm->code
$11 = (int **) 0x555befd712a0
(gdb)

we use locals in order to write vm->globals.

&vm->globals == vm->call_stack[callsp].locals[(0x555befd712b0-0x555befd72234)/4]
&vm->code == vm->call_stack[callsp].locals[(0x555befd712a0-0x555befd72234)/4]

// address of locals is afterwards as it is initialize afterwards


Before reaching to ret instruction there is 
free which segfault if we have already inputted
random data in it. so can store address of vm->globals
before overwriting it and then restore address afterwards
before calling free.

In asd.vm that's what it first two instruction 
are doing storing address of vm->globals in virtual machines 
stack.

// here, also There is number line concept where we are on the 0 position
// We wanna read next four byte of vm->globals so we are comming towards 
// zero.

other things about calling function and ropping are discussed
in asd.vm.

> (cat asd; cat) | nc 172.17.0.2 1337
> /bin/cat /flag

