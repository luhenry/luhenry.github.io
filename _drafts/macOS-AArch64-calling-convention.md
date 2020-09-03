---
title: "macOS-AArch64: Calling Convention"
layout: post
---

_This is a series about some discoveries I make in adding support for macOS-AArch64 to the OpenJDK_

## Understanding the problem

While trying to make `java -version` run, I encountered the following issue:

```
$> build/macosx-aarch64-server-slowdebug/jdk/bin/java -version
Error occurred during initialization of boot layer
java.lang.InternalError: DMH.invokeStatic=Lambda(a0:L,a1:L,a2:L,a3:L,a4:L,a5:L,a6:L)=>{
    t7:L=DirectMethodHandle.internalMemberName(a0:L);
    t8:L=MethodHandle.linkToStatic(a1:L,a2:L,a3:L,a4:L,a5:L,a6:L,t7:L);t8:L}
Caused by: java.lang.IllegalArgumentException: classData is only applicable for hidden classes
```

From a quick search for `classData is only applicable for hidden classes`, we find that the exception is thrown from [src/hotspot/share/prims/jvm.cpp:1025](https://github.com/openjdk/jdk/blob/869b05169fdb3a1ac851b367a2284ca0c5bb4d7a/src/hotspot/share/prims/jvm.cpp#L1025).

Let's run with a debugger to gather more information about the bug:

```
$> lldb -- build/macosx-aarch64-server-slowdebug/jdk/bin/java -version
(lldb) target create "build/macosx-aarch64-server-slowdebug/jdk/bin/java"
Current executable set to '/Users/luhenry/openjdk-jdk/build/macosx-aarch64-server-slowdebug/jdk/bin/java' (arm64).
(lldb) settings set -- target.run-args  "-version"
(lldb) b jvm.cpp:1025
Breakpoint 1: no locations (pending).
WARNING:  Unable to resolve breakpoint to any actual locations.
(lldb) r
Process 61939 launched: '/Users/luhenry/openjdk-jdk/build/macosx-aarch64-server-slowdebug/jdk/bin/java' (arm64)
1 location added to breakpoint 1
Process 61939 stopped
* thread #3, stop reason = breakpoint 1.1
    frame #0: 0x000000010604df0c libjvm.dylib`jvm_lookup_define_class(env=0x0000000100816ba8, lookup=0x000000017008c7a0, name="java/lang/invoke/LambdaForm$DMH", buf=0x000000010501b000, len=1212, pd=0x0000000000000000, init='\x01', flags=0, classData=0x000000000000000a, __the_thread__=0x0000000100816820) at jvm.cpp:1025:7
   1022   if (!is_hidden) {
   1023     // classData is only applicable for hidden classes
   1024     if (classData != NULL) {
-> 1025       THROW_MSG_0(vmSymbols::java_lang_IllegalArgumentException(), "classData is only applicable for hidden classes");
   1026     }
   1027     if (is_nestmate) {
   1028       THROW_MSG_0(vmSymbols::java_lang_IllegalArgumentException(), "dynamic nestmate is only applicable for hidden classes");
Target 0: (java) stopped.
```

We have, `is_hidden = (flags & HIDDEN_CLASS) == HIDDEN_CLASS`, which is false with `flags = 10`. However, `classData` has an unexpected value: `0xa`. It is indeed non-NULL, which is why it throws an exception, but we expect either a NULL value or a valid pointer to a Java object. Here, `0xa` is neither of these.

Let's backtrack a bit and figure out where these values come from.

First, let's take a look at the backtrace:

```
(lldb) bt
* thread #3, stop reason = breakpoint 1.1
  * frame #0: 0x000000010604df0c libjvm.dylib`jvm_lookup_define_class(env=0x0000000100816ba8, lookup=0x000000017008c7a0, name="java/lang/invoke/LambdaForm$DMH", buf=0x000000010501b000, len=1212, pd=0x0000000000000000, init='\x01', flags=0, classData=0x000000000000000a, __the_thread__=0x0000000100816820) at jvm.cpp:1025:7
    frame #1: 0x000000010604dbb0 libjvm.dylib`::JVM_LookupDefineClass(env=0x0000000100816ba8, lookup=0x000000017008c7a0, name="java/lang/invoke/LambdaForm$DMH", buf=0x000000010501b000, len=1212, pd=0x0000000000000000, initialize='\x01', flags=0, classData=0x000000000000000a) at jvm.cpp:1139:10
    frame #2: 0x0000000100502cfc libjava.dylib`Java_java_lang_ClassLoader_defineClass0(env=0x0000000100816ba8, cls=0x000000017008c758, loader=0x0000000000000000, lookup=0x000000017008c7a0, name=0x000000017008c798, data=0x000000017008c790, offset=0, length=1212, pd=0x0000000000000000, initialize='\x01', flags=0, classData=0x000000000000000a) at ClassLoader.c:263:12
    frame #3: 0x0000000108080aa0
    frame #4: 0x000000010807bde0
[...]
```

We can see that the value of `classData` comes without modification from `Java_java_lang_ClassLoader_defineClass0`. Looking further into this function, we note that it is the native implementation of `java.lang.ClassLoader.defineClass0` (see [src/java.base/share/classes/java/lang/ClassLoader.java:1134](https://github.com/openjdk/jdk/blob/869b05169fdb3a1ac851b367a2284ca0c5bb4d7a/src/java.base/share/classes/java/lang/ClassLoader.java#L1134)).

Next, let's verify what values Java is passing:

```
--- a/src/java.base/share/classes/java/lang/System.java
+++ b/src/java.base/share/classes/java/lang/System.java
@@ -2190,6 +2190,7 @@ public final class System {
             }
             public Class<?> defineClass(ClassLoader loader, Class<?> lookup, String name, byte[] b, ProtectionDomain pd,
                                         boolean initialize, int flags, Object classData) {
+                System.err.println("ClassLoader.defineClass0(" + loader + ", " + lookup + ", " + name + ", " + b + ", " + 0 + ", " + b.length + ", " + pd + ", " + initialize + ", " + flags + ", " + classData + ")");
                 return ClassLoader.defineClass0(loader, lookup, name, b, 0, b.length, pd, initialize, flags, classData);
             }
             public Class<?> findBootstrapClassOrNull(ClassLoader cl, String name) {
```

```
$> lldb -- build/macosx-aarch64-server-slowdebug/jdk/bin/java -version
(lldb) target create "build/macosx-aarch64-server-slowdebug/jdk/bin/java"
Current executable set to '/Users/luhenry/openjdk-jdk/build/macosx-aarch64-server-slowdebug/jdk/bin/java' (arm64).
(lldb) settings set -- target.run-args  "-version"
(lldb) b Java_java_lang_ClassLoader_defineClass0
Breakpoint 1: no locations (pending).
WARNING:  Unable to resolve breakpoint to any actual locations.
(lldb) r
Process 64011 launched: '/Users/luhenry/openjdk-jdk/build/macosx-aarch64-server-slowdebug/jdk/bin/java' (arm64)
1 location added to breakpoint 1
Process 64011 stopped
ClassLoader.defineClass0(null, class java.lang.invoke.LambdaForm, java/lang/invoke/LambdaForm$DMH, [B@7e0b37bc, 0, 1212, null, true, 10, [DMH.invokeStatic=Lambda(a0:L,a1:L,a2:L,a3:L,a4:L,a5:L,a6:L)=>{
    t7:L=DirectMethodHandle.internalMemberName(a0:L);
    t8:L=MethodHandle.linkToStatic(a1:L,a2:L,a3:L,a4:L,a5:L,a6:L,t7:L);t8:L}])
Process 64011 stopped
* thread #3, stop reason = breakpoint 1.1
    frame #0: 0x0000000100502bbc libjava.dylib`Java_java_lang_ClassLoader_defineClass0(env=0x0000000100816ba8, cls=0x000000017008c758, loader=0x0000000000000000, lookup=0x000000017008c7a0, name=0x000000017008c798, data=0x000000017008c790, offset=0, length=1212, pd=0x0000000000000000, initialize='\x01', flags=0, classData=0x000000000000000a) at ClassLoader.c:226:12
   223  {
   224      jbyte *body;
   225      char *utfName;
-> 226      jclass result = 0;
   227      char buf[128];
   228
   229      if (data == NULL) {
Target 0: (java) stopped.
```

We note a few things:
 - `flags` is equal to `10` in Java but `0` in native
 - `classData` is a valid, non-NULL object in Java, but it is equal to `0xa` in native.

The value of `classData` in native is the last clue we need: `0xa` is the hexadecimal representation of `10`. That means that while Java is passing `flags = 10` and `classData = <valid jobject>`, the native callee receives `flags = 0` and `classData = 10`.

It is a classic example of a calling convention mismatch between the caller and the callee. On the one hand, the caller, respecting a specific ABI[^1], puts the parameters in a pre-defined set of locations (register or stack slots). On the other hand, the callee, respecting another ABI, expects the parameters to be passed in a different pre-defined set of locations.

## Understanding the ABI

I am using [godbolt.org](https://godbolt.org) and the following C snippet to explore more simply the native calling convention.

```
void func(int64_t p0, int64_t p1, int64_t p2, int64_t p3, int64_t p4, int64_t p5, int32_t p6, int32_t p7, int64_t p8, bool p9, int32_t p10, int64_t p11);
```

### The native calling convention

Let's start by the native side to understand where it expects its parameters to be passed, because it is the native C/C++ compiler compiling the `Java_java_lang_ClassLoader_defineClass0` function.

From analyzing the generated assembly, the parameters are expected as follows:
 - `p0`, `p1`, `p2`, `p3`, `p4`, `p5`, `p6` and `p7` are read from registers in `x0`, `x1`, `x2`, `x3`, `x4`, `x5`, `w6`, and `w7` respectively
 - `p8`, `p9`, `p10` and `p11` are read from the stack at `sp+0`, `sp+8`, `sp+12` and `sp+16` respectively

That is in line with the [official documentation](https://developer.apple.com/library/archive/documentation/Xcode/Conceptual/iPhoneOSABIReference/Articles/ARM64FunctionCallingConventions.html#//apple_ref/doc/uid/TP40013702-SW4) of macOS-AArch64 provided by Apple. (This link is for the iOS-AArch64 ABI since, per [Addressing Architectural Differences in Your macOS Code](https://developer.apple.com/documentation/apple_silicon/addressing_architectural_differences_in_your_macos_code) > "Update C++ Code", the macOS-AArch64 ABI matches the iOS-AArch64 ABI.)

### The Java on AArch64 calling convention

Now, let's check how Hotspot passes parameters. (Since we have access to the sources, we do not need to look at the generated assembly. Yeah!)

This native function can be called either by the interpreter or by compiled code. In our case, we know that it is the interpreter which invokes `java.lang.ClassLoader.defineClass0`. We then focus on how the interpreter invokes native methods.

From looking at [src/hotspot/cpu/aarch64/interpreterRT_aarch64.cpp:89](https://github.com/openjdk/jdk/blob/869b05169fdb3a1ac851b367a2284ca0c5bb4d7a/src/hotspot/cpu/aarch64/interpreterRT_aarch64.cpp#L89), we notice that the arguments passed on the stack are 8-bytes aligned. That's our problem!

For a recap of where parameters are passed from Hotspot:
 - `p0`, `p1`, `p2`, `p3`, `p4`, `p5`, `p6` and `p7` are written to registers in `x0`, `x1`, `x2`, `x3`, `x4`, `x5`, `w6`, and `w7` respectively
 - `p8`, `p9`, `p10` and `p11` are written to the stack at `sp+0`, `sp+8`, `sp+16` and `sp+24` respectively

Let's map that in a table. (Note that the memory ordering is little-endian.)

```
sp+0     sp+4     sp+8     sp+12    sp+16    sp+20    sp+24    sp+28
00000000 00000000 10000000 00000000 a0000000 00000000 022d1a9f 10000000
^ pd              ^ init            ^ flags           ^ classData       // Java ABI
^ pd              ^ init   ^ flags  ^ classData                         // native ABI
```

From that, we can see why `flags` is `10` in Java an `0` in native, and why `classData` is a valid pointer in Java but `0xa` in native.

## How to fix it?

You may ask, why in the heck is OpenJDK passing arguments like that? The answer might be quite disappointing: because it is the default ABI on AArch64, and it is, in fact, macOS-AArch64 that deviates from the norm. It is also why there is no such issue in the port of Hotspot to Linux-AArch64 and Windows-AArch64.

In `void InterpreterRuntime::SignatureHandlerGenerator::pass_int();`, we have the following:

```
void InterpreterRuntime::SignatureHandlerGenerator::pass_int() {
  const Address src(from(), Interpreter::local_offset_in_bytes(offset()));

  switch (_num_int_args) {
  case 0:
    __ ldr(c_rarg1, src);
    _num_int_args++;
    break;
  case 1:
    __ ldr(c_rarg2, src);
    _num_int_args++;
    break;

[...]

  default:
    __ ldr(r0, src);
    __ str(r0, Address(to(), _stack_offset));
    _stack_offset += wordSize;
    _num_int_args++;
    break;
  }
}
```

The `default` case is for any parameter passed on the stack. The solution is then to make sure that the `_stack_offset` for the next parameter is not 8-bytes aligned, but 4-bytes aligned in the case of an `int`.

The fix is the following:

```
--- a/src/hotspot/cpu/aarch64/interpreterRT_aarch64.cpp
+++ b/src/hotspot/cpu/aarch64/interpreterRT_aarch64.cpp
@@ -86,7 +86,7 @@ void InterpreterRuntime::SignatureHandlerGenerator::pass_int() {
   default:
     __ ldr(r0, src);
     __ str(r0, Address(to(), _stack_offset));
-    _stack_offset += wordSize;
+    _stack_offset += MACOS_ONLY(4) NOT_MACOS(wordSize);
     _num_int_args++;
     break;
   }
```

However, we still need to make sure that any 8-bytes wide values (like `long long`, objects or pointers in general) are still 8-bytes aligned:

```
--- a/src/hotspot/cpu/aarch64/interpreterRT_aarch64.cpp
+++ b/src/hotspot/cpu/aarch64/interpreterRT_aarch64.cpp
@@ -125,6 +125,7 @@ void InterpreterRuntime::SignatureHandlerGenerator::pass_long() {
     _num_int_args++;
     break;
   default:
+    _stack_offset = align_up(_stack_offset, 8);
     __ ldr(r0, src);
     __ str(r0, Address(to(), _stack_offset));
     _stack_offset += wordSize;
```

With these fixes, it now runs successfully:

```
$> build/macosx-aarch64-server-slowdebug/jdk/bin/java -version
openjdk version "16-internal" 2021-03-16
OpenJDK Runtime Environment (slowdebug build 16-internal+0-adhoc.luhenry.openjdk-jdk)
OpenJDK 64-Bit Server VM (slowdebug build 16-internal+0-adhoc.luhenry.openjdk-jdk, mixed mode)
```

[^1]: [Application Binary Interface](https://en.wikipedia.org/wiki/Application_binary_interface)
