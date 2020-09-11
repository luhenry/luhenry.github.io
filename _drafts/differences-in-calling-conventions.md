---
title: "Differences in Calling Conventions"
layout: post
---

_This is an installment in a [series of posts]({% post_url 2020-09-07-openjdk-on-aarch64 %}) that will highlight discoveries I am making as I add support for Windows-AArch64 and macOS-AArch64 to the OpenJDK_

In the [ARM64 Function Calling Convention](https://developer.apple.com/library/archive/documentation/Xcode/Conceptual/iPhoneOSABIReference/Articles/ARM64FunctionCallingConventions.html), Apple describes where and how the macOS-AArch64 calling convention differs from the [official one](https://developer.arm.com/documentation/ihi0055/b/) used on Linux and Windows. This calling convention is part of the ABI which you can read more about at [What's an ABI anyways?]({% post_url 2020-09-08-whats-an-abi-anyway %}).

In the official calling convention, parameters are 8-bytes aligned, while on macOS (and iOS), the parameters are aligned on their size. For example, `int` is 4-bytes wide and 4-bytes aligned, and `short` is 2-bytes wide and 2-bytes aligned. That impacts any Java code calling into native code (into the VM or via JNI, for example). We can expose this failures with something as simple as `java -version`.

## The symptoms

That is what happens when I run `java -version`:

```
$> build/macosx-aarch64-server-slowdebug/jdk/bin/java -version
Error occurred during initialization of boot layer
java.lang.InternalError: DMH.invokeStatic=Lambda(a0:L,a1:L,a2:L,a3:L,a4:L,a5:L,a6:L)=>{
    t7:L=DirectMethodHandle.internalMemberName(a0:L);
    t8:L=MethodHandle.linkToStatic(a1:L,a2:L,a3:L,a4:L,a5:L,a6:L,t7:L);t8:L}
Caused by: java.lang.IllegalArgumentException: classData is only applicable for hidden classes
```

From a quick search in the OpenJDK source code for `classData is only applicable for hidden classes`, we find that the exception is thrown from [src/hotspot/share/prims/jvm.cpp:1025](https://github.com/openjdk/jdk/blob/869b05169fdb3a1ac851b367a2284ca0c5bb4d7a/src/hotspot/share/prims/jvm.cpp#L1025).

Running with a debugger yielded more information about the crash:

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

We have, `is_hidden = (flags & HIDDEN_CLASS) == HIDDEN_CLASS`, which is false with `flags = 10`. However, `classData` has an unexpected value: `0xa`. It is indeed non-NULL, which is why it throws an exception, but we expect either a NULL value or a valid pointer to a Java object. Here, `0xa` is neither of those.

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

We can see that the value of `classData` comes straight from `Java_java_lang_ClassLoader_defineClass0`. Looking further into this function, we note that it is the native implementation of `java.lang.ClassLoader.defineClass0` (see [src/java.base/share/classes/java/lang/ClassLoader.java:1134](https://github.com/openjdk/jdk/blob/869b05169fdb3a1ac851b367a2284ca0c5bb4d7a/src/java.base/share/classes/java/lang/ClassLoader.java#L1134)).

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

Here is what we have learned so far:
- `flags` is equal to `10` in Java but `0` in native
- `classData` is a valid, non-NULL object in Java, but it is equal to `0xa` in native.

This is a classic example of a calling convention mismatch between the caller and the callee. On the one hand, the caller, respecting a specific ABI, puts the parameters in a pre-defined set of locations (register or stack slots). On the other hand, the callee, respecting another ABI, expects the parameters to be passed in a different pre-defined set of locations.

## Understanding the differences in ABI

Let's visualize the differences between the calling conventions of Linux-AArch64 and macOS-AArch64.

| Parameter | Size (bytes) | Linux-AArch64 | macOS-AArch64 |
| --------- | -----------: | ------------: | ------------: |
| `env`     | 8            | `r0`          | `r0`          |
| `cls`     | 8            | `r1`          | `r1`          |
| `loader`  | 8            | `r2`          | `r2`          |
| `lookup`  | 8            | `r3`          | `r3`          |
| `name`    | 8            | `r4`          | `r4`          |
| `data`    | 8            | `r5`          | `r5`          |
| `offset`  | 4            | `r6`          | `r6`          |
| `length`  | 4            | `r7`          | `r7`          |
| `pd`      | 8            | `sp+0`        | `sp+0`        |
| `initialize` | 1         | `sp+8`        | `sp+8`        |
| `flags`   | 4            | **`sp+16`**   | **`sp+12`**   |
| `classData` | 8          | **`sp+24`**   | **`sp+16`**   |

You notice the difference between Linux-AArch64 and macOS-AArch64 around `flags` and `classData`.

Hotspot currently follows the Linux-AArch64 calling convention while native follows the macOS-AArch64 calling convention.

Let's map the stack at the time of the call. _(Note that the memory ordering is little-endian.)_

```
                     Java         native
sp+28 | 10000000 |             
sp+24 | 022d1a9f | < classData
sp+20 | 00000000 |            
sp+16 | a0000000 | < flags      < classData
sp+12 | 00000000 |              < flags
sp+8  | 10000000 | < init       < init
sp+4  | 00000000 |            
sp+0  | 00000000 | < pd         < pd
```

This clarifies why `flags` is `10` in Java but `0` in native, and why `classData` is a valid pointer in Java but `0xa` in native.

## How to fix it?

The fix is to teach Hotspot to use the macOS-AArch64 calling convention when running on macOS-AArch64.

Luckily there are only a few places in Hotspot that generate this transition from Java to native: in the interpreter and in the compiler. Due to technical and historical reasons, the code is not shared across these two. We'll then need to modify both for everything to work.

In [`InterpreterRuntime::SignatureHandlerGenerator::pass_int`](https://github.com/openjdk/jdk/blob/869b05169fdb3a1ac851b367a2284ca0c5bb4d7a/src/hotspot/cpu/aarch64/interpreterRT_aarch64.cpp#L54-L93), we have the following:

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

  default: // for any parameter passed on the stack
    __ ldr(r0, src);
    __ str(r0, Address(to(), _stack_offset));
    _stack_offset += wordSize;
    _num_int_args++;
    break;
  }
}
```

The solution is to ensure that the `_stack_offset` for the next parameter is not 8-bytes aligned, but 4-bytes aligned for an `int`.

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

However, we still need to ensure that any 8-bytes wide values (like `long`, objects, or pointers in general) are still 8-bytes aligned.

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

With these fixes and a few others similar to this one, `java -version` now runs successfully:

```
$> build/macosx-aarch64-server-slowdebug/jdk/bin/java -version
openjdk version "16-internal" 2021-03-16
OpenJDK Runtime Environment (slowdebug build 16-internal+0-adhoc.luhenry.openjdk-jdk)
OpenJDK 64-Bit Server VM (slowdebug build 16-internal+0-adhoc.luhenry.openjdk-jdk, mixed mode)
```

## Conclusion

We explored how the macOS-AArch64 ABI differs from the Linux-AArch64 ABI, and its impact on Java to native method calls. We also explored what modifications are necessary for Hotspot to match the different calling conventions between macOS, Linux, and Windows.

In later posts, I'll talk more about some of the issues I ran into when porting the OpenJDK to Windows-AArch64 and macOS-AArch64, the subtle differences in their ABI and APIs, and the necessary modifications to the OpenJDK.
