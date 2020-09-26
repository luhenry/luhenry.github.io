---
title: "From stack frames to stack traces"
layout: post
---

_This is an installment in a [series of posts]({% post_url 2020-09-07-openjdk-on-aarch64 %}) that will highlight discoveries I am making as I add support for Windows-AArch64 and macOS-AArch64 to the OpenJDK._

Whenever a function is called, a stack frame is allocated. This stack frame will contain information like local variables, arguments passed to a callee, or values spilled from registers. But how exactly are stack frames setup so that you can walk through them? That is what we are going to explore in this post.

In [What's an ABI, anyways?]({% post_url 2020-09-08-whats-an-abi-anyways %}) and [Differences in Calling Conventions]({% post_url 2020-09-14-differences-in-calling-conventions %}), we saw that this can change across platforms. To keep it simple we focus in this post on the Linux-AArch64 ABI.

## Linking stack frames

A stack trace in itself is a linked list of stack frames, with each frame having a pointer to its parent. If we want to express it in terms of C structure, it would be defined as follows:

```
typedef struct frame { 
    intptr_t[N]   data;
    struct frame *fp; // the caller's stack frame address
    void         *lr; // the caller's return address
} frame;
```

In this structure, `N` is a per-function constant that is determined by the compiler. It is big enough that it can contain all the values necessary to store on the stack, but as small as possible to avoid wasting stack space. 

You can then, for example, have the following on the stack:

```
     0x7200 |             |
            |-------------|
     0x71f8 | lr = 0x0    |<-\ // the thread's root frame, it has no parent
     0x71f0 | fp = 0x0    |  |
     [...]  | ...         |  |
     0x71d0 | data[0]     |  |
            |-------------|  |
     0x71c8 | lr = 0x12   |  |
     0x71c0 | fp = 0x7200 |<-\
     [...]  | ...         |  |
     0x71e8 | data[0]     |  |
            |-------------|  |
     0x71e0 | lr = 0x5a   |  |
     0x71d8 | fp = 0x71d0 |<-\
            |-------------|  |
     0x70d0 | lr = 0x90   |  |
     0x70c8 | fp = 0x71e8 |<-\
     0x70c0 | data[2]     |  |
     0x70b8 | data[1]     |  |
rfp: 0x70b0 | data[0]     |  |
            |-------------|  |
     0x70ac | lr = 0x42   |  | // the thread's currently executing frame
     0x70a0 | fp = 0x70d8 |--/
     0x709c | data[3]     |
     0x7090 | data[2]     |
     0x708c | data[1]     |
rsp: 0x7080 | data[0]     |
            |-------------|
```

That raises some important questions:

_How does the thread knows where the current frame is?_ The pointer to the current stack frame is stored in the Frame Pointer (FP) register (`r30`/`rfp`). In the example above, `rfp = 0x70b0`.

_How does the currently executing function access local variables?_ It could use the FP register, but it doesn't. Instead, it uses the Stack Pointer (SP) register (`r31`/`rsp`). The difference between the SP and FP registers is simple: the FP register points to the top of the frame, and the SP register points to the bottom of the frame. And in the case of the SP register, it simply points to the `frame->data[0]`. In the example above, `rsp = 0x7080`.

_Why is the stack frame at `0x71d8-0x71e0` empty?_ It can happen that a function never needs to use the stack. That's especially true on AArch64 because of the large number of registers which reduce the need to spill or store data out of registers and onto the stack.

_Hey, didn't you get the ordering wrong?_ Indeed, you could say that it's weird that the stack seems to grow downward. However, that is simply the convention on ARM and x86, and on major OS like Linux, Windows, and macOS. 

## Setting up and cleaning up a frame

Now that we understand what the stack frame looks like, let's dive into how the generated code allocates it.

Because a stack frame needs to be allocated for every function call (expect for inlined functions and tail call, topics which are beyond this post), the operation needs to be fast.

The sequence of AArch64 instructions is the following:

```
stp     rfp, lr, [rsp - 2 * wordSize]
mov     rfp, rsp
sub     rsp, rsp, N * wordSize // Only required if values need to be stored on the stack
```


