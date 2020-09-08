---
title: "What's an ABI anyway?"
layout: post
---

_This post is part a [series of blog posts]({% post_url 2020-09-07-openjdk-on-aarch64 %}) about some discoveries I make in adding support for Windows-AArch64 and macOS-AArch64 to the OpenJDK_

Before we dive further into specifics of Windows-AArch64 and macOS-AArch64, it's essential to lay out some of a platform's fundamental concepts. Here, I'm diving deeper into what an ABI is.

## Definition

ABI stands for Application Binary Interface. It sounds very similar to API (Application Programming Interface) because it is, in fact, a very similar concept.

Let's take an example of a C function defined in a header file:

```
// my_library.h
int my_func(long p0, long p1, long p2, long p3, long p4, long p5, int p6, int p7, long p8, char p9, int p10, long p11);
```

To invoke this function from another file, you do the following:

```
// my_exe.c
#include "my_library.h"
int main(int argc, char **argv) {
  return my_func(0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11);
}
```

Because you declared the signature of the function `my_func`, the compiler will know how to generate the code to call it. It will know how many parameters there are, the type of each parameter, and the return type. With that, it knows whether you're invoking the method correctly:
 - If you try to pass more or less than 12 parameters, it will fail to compile
 - If you try to pass a `long` instead of a `char`, it will fail to compile
 - If you ignore the return value, it may emit a warning

The API is then a contract between the caller and the callee for functions and data types. (This is the same for classes and methods in Object-Oriented Programming, for classes and their methods.)

After the compiler validates that a caller invokes a function the way the callee expects it, it generates the assembly code that does the call. That is where the ABI comes into play.

Overall, an ABI is similar to an API: it defines a set of conventions. The API does it at the source-level, and the ABI does it at the assembly-level.

Parts of what the ABI defines are:
 - The **calling convention**, or "how to invoke a function": the instructions to emit, how to pass parameters, how to set up the stack, which way to allocate the stack (up or down)
 - The **size and alignment of basic data types**, like `int`, `short`, `long`, `long long` or pointers
 - The **usage of machine registers**, like `r31` to store the stack pointer, or `r30` to store the caller's return address (also called the "link register" on ARM)

It is also specific to each platform: Windows vs. Linux vs. macOS, ARM vs. x86, 32-bits vs. 64-bits.

The most common ABI is the C ABI. Following it allows different languages (each with a specific compiler) to call each other's functions. For example, the Java compiler knows nothing about C/C++ headers. Still, by defining the equivalent signature in Java to the C function, it will know where to put the parameters, the size of each parameter, and where to get the return value, by merely following the ABI.

### An analogy

Imagine an assembly line where two robots are collaborating to build an end-product. Robot-1 puts parts into five boxes, waits for Robot-2 to do its job, picks up the end-product from the "return" box, and stores it elsewhere. Robot-2 takes the parts from the five boxes, assembles them, and drops the end-product back into the "return" box. Unfortunately, each robot has no way to communicate other than through these boxes, and you can reconfigure these boxes only when you're re-designing the assembly line.

For a smooth operation, both robots need to agree on which box to put each part. If at any point, Robot-1 starts putting parts into other boxes, or Robot-2 takes a part from a different box, Robot-2 can miss some of the necessary parts or end up assembling the wrong end-product.

The ABI defines the convention that allows both robots to work effectively as each knows, for example, which box to use for which part and each part's size in each box. It also makes it possible to drop-in a new robot, as long as it respects where and how each part goes in each box. This new robot then doesn't even need to come from the same manufacturer or be programmed the same way, as long as it respects each part's convention to come from specific boxes.

## Example

Let's take `my_func` above and focus on the Linux-AArch64 ABI (the combination of Linux/ARM/64-bits).

Per [the official documentation](https://developer.arm.com/documentation/ihi0055/b/), the ABI defines:
 - A `char` is 1-byte wide, an `int` is 4-bytes wide, and a `long` is 8-bytes wide
 - The first 8 parameters are passed in registers `r0...r7`
 - The next 4 parameters are passed on the stack with an 8-bytes alignment
 - The return value is passed in `r0`

For `my_func`, the following illustrate where arguments are passed in registers and the stack. (Note that the memory ordering is little-endian.)

```
// parameters passed in registers
r0: 00000000 00000000    < p0 and return value
r1: 10000000 00000000    < p1
r2: 20000000 00000000    < p2
r3: 30000000 00000000    < p3
r4: 40000000 00000000    < p4
r5: 50000000 00000000    < p5
r6: 60000000 00000000    < p6
r7: 70000000 00000000    < p7

// parameters passed on the stack
sp+0:  80000000 00000000 < p8
sp+8:  90000000 00000000 < p9
sp+16: a0000000 00000000 < p10
sp+24: b0000000 00000000 < p11
```

For `main` to call `my_func`, `main` then needs to put the parameters in these predefined locations. Failing to do so leads `my_func` to look for parameters where they haven't been passed. That can lead (at best) to a crash, or (at worst) to the application's state's silent corruption.

## Conclusion

We learned what an ABI is and why it is an essential component of a platform. We also explored some of the calling convention aspects of the ABI on Linux-AArch64.

In later posts, I'll talk about some of the issues I ran into when porting the OpenJDK to Windows-AArch64 and macOS-AArch64, the subtle differences in their ABI, and the necessary modifications to the OpenJDK.
