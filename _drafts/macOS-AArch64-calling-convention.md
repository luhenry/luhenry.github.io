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

We have, `is_hidden = (flags & HIDDEN_CLASS) == HIDDEN_CLASS`, which is false with `flags = 10`. However, `classData` has an unexpected value: `0xA`. It is indeed non-NULL, which is why it throws an exception, but we expect either a NULL value or a valid pointer to a Java object. Here, `0xA` is neither of these.

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
diff --git a/src/java.base/share/classes/java/lang/System.java b/src/java.base/share/classes/java/lang/System.java
index 4bc6f30d473..fbe027428e3 100644
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
 - `classData` is a valid, non-NULL object in Java, but it is equal to `0xA` in native.

The value of `classData` in native is the last clue we need: `0xA` is the hexadecimal representation of `10`. That means that while Java is passing `flags = 10` and `classData = <valid jobject>`, the native callee receives `flags = 0` and `classData = 10`.

It is a classic example of a calling convention mismatch between the caller and the callee. On the one hand, the caller, respecting a specific ABI[^1], puts the parameters in a pre-defined set of locations (register or stack slots). On the other hand, the callee, respecting another ABI, expects the parameters to be passed in a different pre-defined set of locations.

## Understanding the ABI

I am using [godbolt.org](https://godbolt.org) and the following C snippet to simplify further explorations.

```
void func(int p0, int p1, int p2, int p3, int p4, int p5, int p6, int p7, int p8, int p9, int p10, int p11) {
    static volatile int _i;
    _i = p0;
    _i = p1;
    _i = p2;
    _i = p3;
    _i = p4;
    _i = p5;
    _i = p6;
    _i = p7;
    _i = p8;
    _i = p9;
    _i = p10;
    _i = p11;
}
```

We are focusing on the calling convention of this snippet of code.

### The native calling convention

Because it is the native C/C++ compiler compiling the `Java_java_lang_ClassLoader_defineClass0` function, let's start by the native side to understand where it expects its parameters to be passed.

Let's analyze from the generated assembly where parameters are expected:
 - `p0`, `p1`, `p2`, `p3`, `p4`, `p5`, `p6` and `p7` are passed in registers in `w0`[^2], `w1`[^2], `w2`, `w3`, `w4`, `w5`, `w6`, and `w7` respectively
 - `p8`, `p9`, `p10` and `p11` are passed on the stack at `sp[0]`, `sp[4]`, `sp[8]` and `sp[12]` respectively

We note the arguments passed on the stack are 4-bytes aligned as they are of type `int`, which is 4-bytes wide.

That is in line with the [official documentation](https://developer.apple.com/library/archive/documentation/Xcode/Conceptual/iPhoneOSABIReference/Articles/ARM64FunctionCallingConventions.html#//apple_ref/doc/uid/TP40013702-SW4) of macOS-AArch64 provided by Apple. (This link is for the iOS-AArch64 ABI since, per [Addressing Architectural Differences in Your macOS Code](https://developer.apple.com/documentation/apple_silicon/addressing_architectural_differences_in_your_macos_code) > "Update C++ Code", the macOS-AArch64 ABI matches the iOS-AArch64 ABI.)

### The OpenJDK on AArch64 calling convention

Now, let's check where the OpenJDK passes these parameters.

(Since we have access to the sources, we do not need to look at the generated assembly. Yeah!)



[^1]: [Application Binary Interface](https://en.wikipedia.org/wiki/Application_binary_interface)
[^2]: `w0` denotes the 32-bits value stored in register `0`, `w1` denotes the 32-bits value stored in register `1`, etc.
