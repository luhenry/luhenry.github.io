---
title: "From stack frames to stack traces"
layout: post
---

_This is an installment in a [series of posts]({% post_url 2020-09-07-openjdk-on-aarch64 %}) that will highlight discoveries I am making as I add support for Windows-AArch64 and macOS-AArch64 to the OpenJDK._

Whenever a function is called, a stack frame is allocated. This stack frame will contain information like local variables, arguments passed to a callee, or values spilled from registers. But how exactly are stack frames setup so that you can walk through them? That is what we are going to explore in this post.

In [What's an ABI, anyways?]({% post_url 2020-09-08-whats-an-abi-anyways %}) and [Differences in Calling Conventions]({% post_url 2020-09-14-differences-in-calling-conventions %}), we saw that this can change across platforms. To keep it simple let's focus in this post on the Linux-AArch64 and Linux-x86_64 ABIs.

## Defining a stack trace

A stack trace in itself is a linked list of stack frames, with each frame having a pointer to its parent. If we want to express it in terms of C structure, it would be defined as follows for Linux-AArch64:

```
typedef struct frame {
    struct frame *fp;
    void         *lr; // the caller's return address
    byte[]        data;
} frame;
```

- Describe how this frame struct is created when entering a function.

You can then imagine having the following on the stack:

```
       |-------------|
0x71fc | fp = 0x0    |<-\ // the thread's root frame, it has no parent
       | ...         |  | // 40 bytes for local variables or spilled registers
       |-------------|  |
0x71cc | lr = 0x12   |  |
0x71c4 | fp = 0x71fc |<-\
       | ...         |  |
       |-------------|  |
0x718c | lr = 0x5a   |  |
0x7184 | fp = 0x71c4 |<-\
       | ...         |  |
       |-------------|  |
0x70f4 | lr = 0x90   |  |
0x70ec | fp = 0x7184 |<-\
       | ...         |  |
       |-------------|  |
0x70ac | lr = 0x42   |  |
0x70a4 | fp = 0x70ec |<-/ // the thread's current frame, it has no child
0x709c | ...         |
0x7094 | ...         |
0x708c | ...         |
0x7084 | ...         |
       |-------------|
```

How does the thread knows the current frame? The pointer to the stack is stored in `r30` by convention. In the example above, `r30 = 0x70a4`.

Let's define the signature of `add10`, the function we are taking as an example:

```
int add9(int p0, int p1, int p2, int p3, int p4, int p5, int p6, int p7, int p8);

int add10(int p0, int p1, int p2, int p3, int p4, int p5, int p6, int p7, int p8, int p9) {
    return add9(p0, p1, p2, p3, p4, p5, p6, p7, p8) + p9;
}
```

- The size of the frame is known by the callee, but not by the caller
- Arguments are passed in caller's frame
- Stack grows down => up is caller's frame
- 

When compiled, the size of the stack frame is 48 bits:

```
sp+40 | 00000000 00000000
sp+32 | 00000000 00000000
sp+24 | 00000000 00000000
sp+16 | 00000000 00000000
sp+8  | 00000000 00000000
sp+0  | 00000000 00000000
```


