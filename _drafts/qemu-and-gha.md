---
title: "QEMU on Github Actions"
layout: post
---

GHA is great! It’s free (20 runners for open repositories per org), it has a rich ecosystem of actions, the runners are managed by the team at GitHub (updates and security fixes), it integrates to GitHub (releases, pull requests, issues, etc.), and it supports multiple platforms: Linux, windows, and macOS on x86.

However, the set of architectures isn’t as exhaustive as what your users may run on. For example, Java is available on aarch64, armhf, s390x (IBM mainframes), ppc64el, and riscv64. You could add self-hosted runners for each of these platforms, but you lose most of the advantages of GitHub-provided runners.

An alternative to self-hosted runner is to use the GitHub-provided runners with a twist: emulation 🫣 (not as scary as you think).

Let's look into what emulation is and how it works.

# Emulation with QEMU

Emulation allows you to run an application written for an architecture (ex: riscv64) on another architecture (ex: x86). This emulation then takes care of translating from riscv64 or s390x to x86 for example, allowing you to transparently run programs across architectures.

The most commonly used emulation software in the Unix ecosystem is QEMU. It has become the Swiss Army knife of emulation, supporting most architectures (many I have never heard of), making it a necessary tool for any new architecture’s ecosystem bringup.

QEMU supports two modes of execution:
* System emulation: it emulates a machine on which you need to boot a Linux kernel. You then SSH into that machine to run your application. It is similar to launching a VM on your machine.
* User-mode emulation: the kernel is still the host’s one and QEMU emulates the syscalls. Whenever your application makes a syscall, QEMU takes over and acts accordingly, calling into the host kernel where it makes sense, “faking” the syscall otherwise.

The user-mode emulation is the easiest to put in place on CI as you launch your application just like any other process, and QEMU makes sure everything “just works”.

So how do you run your application with QEMU? The most explicit is to run `qemu-riscv64-static myapplication`. Let’s take an example of the OpenJDK compiled for riscv64 and running it on an x86 machine (you can download one from adoptium [here](https://ci.adoptium.net/job/build-scripts/job/jobs/job/evaluation/job/jobs/job/jdk21u/job/jdk21u-evaluation-linux-riscv64-temurin/lastSuccessfulBuild/artifact/workspace/target/OpenJDK21U-jdk_riscv64_linux_hotspot_2023-11-13-16-12.tar.gz))

If you try running it, you’ll get the following:
```
$> /path/to/jdk/bin/java -version
bash: exec format error: /path/to/jdk/bin/java
```

That makes sense given the binary is targeting riscv64 and we are trying to run it on x86. We confirm it’s a riscv64 binary with:
```
$> file /path/to/jdk/bin/java
/path/to/jdk/bin/java: ELF 64-bit LSB pie executable, UCB RISC-V, RVC, double-float ABI, version 1 (SYSV), dynamically linked, interpreter /lib/ld-linux-riscv64-lp64d.so.1, BuildID[sha1]=e4445fabaa78b36248d15f0e6a3652939c1f64c1, for GNU/Linux 4.15.0, stripped
```

Notice the `ELF 64-bit LSB pie executable, UCB RISC-V`.


Now, let’s try running it with QEMU as I mentioned before. First install QEMU:
```
$> sudo apt install qemu-user-static
```

Then, run:
```
$> qemu-riscv64-static /path/to/jdk/bin/java -version
qemu-riscv64-static: Could not open '/lib/ld-linux-riscv64-lp64d.so.1': No such file or directory
```

Damn, what’s happening here? Well, QEMU only knows how to translate assembly from riscv64 to x86 here, it doesn’t know how to load libraries, that’s the role of the dynamic linker! Here QEMU is looking for this dynamic linker at `/lib/ld-linux-riscv64-lp64d.so.1` but it can’t find it. That makes sense, there is no such file on the machine.

So where can we find it and how can we tell QEMU where to find it? Via a sysroot and environment variables of course!

(Please don’t run away! It’s easier to set up than you think, I promise!)

# The sysroot

First, let’s setup a sysroot using debootstrap:
```
$> sudo apt install debootstrap
```

Then create a sysroot for riscv64 with the stock Ubuntu 22.04 content:
```
$> sudo debootstrap --arch=riscv64 --verbose --resolve-deps --components=main,universe jammy sysroot
```

If you check what’s in that sysroot folder, you’ll find everything you have when you have a fresh install of Ubuntu 22.04 on a machine:
```
$> ls -alh sysroot
total 68K
drwxr-xr-x 17 root	root	4.0K Nov 13 16:25 .
drwxr-xr-x 40 ludovic ludovic 4.0K Nov 13 17:04 ..
lrwxrwxrwx  1 root	root   	7 Nov 13 16:20 bin -> usr/bin
drwxr-xr-x  2 root	root	4.0K Oct  9 22:54 boot
drwxr-xr-x  4 root	root	4.0K Nov 13 16:20 dev
drwxr-xr-x 68 root	root	4.0K Nov 13 16:25 etc
drwxr-xr-x  2 root	root	4.0K Oct  9 22:54 home
lrwxrwxrwx  1 root	root   	7 Nov 13 16:20 lib -> usr/lib
drwxr-xr-x  2 root	root	4.0K Nov 13 16:20 media
drwxr-xr-x  2 root	root	4.0K Nov 13 16:20 mnt
drwxr-xr-x  2 root	root	4.0K Nov 13 16:20 opt
drwxr-xr-x  2 root	root	4.0K Oct  9 22:54 proc
drwx------  3 root	root	4.0K Nov 13 16:21 root
drwxr-xr-x 10 root	root	4.0K Nov 13 16:24 run
lrwxrwxrwx  1 root	root   	8 Nov 13 16:20 sbin -> usr/sbin
drwxr-xr-x  2 root	root	4.0K Nov 13 16:20 srv
drwxr-xr-x  2 root	root	4.0K Oct  9 22:54 sys
drwxrwxrwt  3 root	root	4.0K Nov 13 16:25 tmp
drwxr-xr-x 11 root	root	4.0K Nov 13 16:20 usr
drwxr-xr-x 11 root	root	4.0K Nov 13 16:20 var
```

But you’ll also notice that everything in there is riscv64:
```
$> file sysroot/bin/bash
sysroot/bin/bash: ELF 64-bit LSB pie executable, UCB RISC-V, RVC, double-float ABI, version 1 (SYSV), dynamically linked, interpreter /lib/ld-linux-riscv64-lp64d.so.1, BuildID[sha1]=7ed4e703c21cd514edcf8100a05580e75e174735, for GNU/Linux 4.15.0, stripped
```

Now, how do we use that sysroot to run our java for riscv64 binary? Simply tell QEMU where to load files from with `QEMU_LD_PREFIX`:
```
$> QEMU_LD_PREFIX=sysroot qemu-riscv64-static /path/to/jdk/bin/java -version
openjdk version "21" 2023-09-19
OpenJDK Runtime Environment (build 21+35-Ubuntu-1)
OpenJDK 64-Bit Server VM (build 21+35-Ubuntu-1, mixed mode, sharing)
```

And it works! Congratulations, you got a riscv64 binary running on x86, isn’t technology incredible? Go ahead, try it out, run a more complex workload. I frequently run the whole of the OpenJDK or larger applications like Apache Spark, and it works (mostly) flawlessly.

Ok, it’s all a bit tedious to use that `qemu-riscv64-static` all the time. And how does it even work when the process forks and creates children? Well it doesn’t out of the box. Actually it does when you install qemu-user-static because it’s smart, but let’s figure out how it's done exactly.

# Binary format

First, let me show you some magic. Go ahead, try the following:
```
$> QEMU_LD_PREFIX=sysroot /path/to/jdk/bin/java -version
openjdk version "21" 2023-09-19
OpenJDK Runtime Environment (build 21+35-Ubuntu-1)
OpenJDK 64-Bit Server VM (build 21+35-Ubuntu-1, mixed mode, sharing)
```

Wait! It Just Works™?? What’s that magic!? Welcome to the wonderful world of binfmt (binary format).

The kernel knows how to load a variety of formats: ELF, a.out, static executables among others. But it's not feasible for the kernel to know all executables format out there, especially as you can get pretty creative - want to execute a JAR file or python script as a plain old executable without prefixing with `java` or `python`, of couse you can do that!

The [binfmt_misc](https://en.m.wikipedia.org/wiki/Binfmt_misc) mechanism allows to add support for these additional mechanism. It allows you to register to the kernel specific "interpreter" for "arbitrary executable file formats to be recognized and passed to certain user space applications, such as emulators and virtual machines." (Read [How programs get run](https://lwn.net/Articles/630727/) and [How programs get run: ELF binaries](https://lwn.net/Articles/631631/) articles on LWN from David Drysdale for in-depth details)

You can find the registered interpreters at `/proc/sys/fs/binfmt_misc`. On my machine I have:
```
$> ls -alh /proc/sys/fs/binfmt_misc
total 0
drwxr-xr-x 2 root root 0 Oct 24 12:26 .
dr-xr-xr-x 1 root root 0 Oct 24 12:26 ..
-rw-r--r-- 1 root root 0 Oct 24 12:26 jar
-rw-r--r-- 1 root root 0 Oct 24 12:26 llvm-10-runtime.binfmt
-rw-r--r-- 1 root root 0 Oct 24 12:26 llvm-11-runtime.binfmt
-rw-r--r-- 1 root root 0 Oct 24 12:26 llvm-14-runtime.binfmt
-rw-r--r-- 1 root root 0 Oct 24 12:26 python3.10
-rw-r--r-- 1 root root 0 Nov 13 17:06 qemu-aarch64
-rw-r--r-- 1 root root 0 Nov 13 17:06 qemu-alpha
-rw-r--r-- 1 root root 0 Nov 13 17:06 qemu-arm
-rw-r--r-- 1 root root 0 Nov 13 17:06 qemu-armeb
-rw-r--r-- 1 root root 0 Nov 13 17:06 qemu-cris
-rw-r--r-- 1 root root 0 Nov 13 17:06 qemu-hexagon
-rw-r--r-- 1 root root 0 Nov 13 17:06 qemu-hppa
-rw-r--r-- 1 root root 0 Nov 13 17:06 qemu-m68k
-rw-r--r-- 1 root root 0 Nov 13 17:06 qemu-microblaze
-rw-r--r-- 1 root root 0 Nov 13 17:06 qemu-mips
-rw-r--r-- 1 root root 0 Nov 13 17:06 qemu-mips64
-rw-r--r-- 1 root root 0 Nov 13 17:06 qemu-mips64el
-rw-r--r-- 1 root root 0 Nov 13 17:06 qemu-mipsel
-rw-r--r-- 1 root root 0 Nov 13 17:06 qemu-mipsn32
-rw-r--r-- 1 root root 0 Nov 13 17:06 qemu-mipsn32el
-rw-r--r-- 1 root root 0 Nov 13 17:06 qemu-ppc
-rw-r--r-- 1 root root 0 Nov 13 17:06 qemu-ppc64
-rw-r--r-- 1 root root 0 Nov 13 17:06 qemu-ppc64le
-rw-r--r-- 1 root root 0 Nov 13 17:06 qemu-riscv32
-rw-r--r-- 1 root root 0 Nov 13 16:08 qemu-riscv64
-rw-r--r-- 1 root root 0 Nov 13 17:06 qemu-s390x
-rw-r--r-- 1 root root 0 Nov 13 17:06 qemu-sh4
-rw-r--r-- 1 root root 0 Nov 13 17:06 qemu-sh4eb
-rw-r--r-- 1 root root 0 Nov 13 17:06 qemu-sparc
-rw-r--r-- 1 root root 0 Nov 13 17:06 qemu-sparc32plus
-rw-r--r-- 1 root root 0 Nov 13 17:06 qemu-sparc64
-rw-r--r-- 1 root root 0 Nov 13 17:06 qemu-xtensa
-rw-r--r-- 1 root root 0 Nov 13 17:06 qemu-xtensaeb
--w------- 1 root root 0 Nov 13 17:06 register
-rw-r--r-- 1 root root 0 Nov 13 17:06 status
```

Let’s look at the interpreter for `qemu-riscv64`:
```
$> cat /proc/sys/fs/binfmt_misc/qemu-riscv64
enabled
interpreter /usr/libexec/qemu-binfmt/riscv64-binfmt-P
flags: POCF
offset 0
magic 7f454c460201010000000000000000000200f300
mask ffffffffffffff00fffffffffffffffffeffffff
```

Here is what we can find:
* The magic number for riscv64
* The path to the qemu-riscv64-static executable
* Some options

Let’s recap how it works:
1. In your shell, you launch your `java` executable compiled for riscv64
2. The shell calls `execve` with the `java` executable as argument
3. The kernel probes for the magic number in the riscv64 `java` executable
4. That magic number matches for the `qemu-riscv64` interpreter
5. The kernel invokes `qemu-riscv64` to “interpret” the riscv64 executable
6. `qemu-riscv64` l then start translating the java executable from riscv64 assembly to x86, and executes that newly generated x86 code

And that’s it! That’s how riscv64 assembly is executed transparently on x86 machines with QEMU.

# Facilitating packaging

The main issue with this whole thing now is that I still need to setup a sysroot, and that’s cumbersome to setup. If only we had a mechanism to ship filesystems around where we can package everything we need and simply run them.

Docker of course! (When is it not a solution?)

The easiest way is the following (assuming `qemu-user-static`` is already setup):
```
$> docker run --rm -it --platform linux/riscv64 riscv64/ubuntu:23.04
```

And with that you will have an Ubuntu 23.04 running riscv64 on your x86 machine 🤯. Isn’t that amazing?

Go ahead, try it. Run `uname -m` for fun. Or even install a package of your choice with `apt install`, it just works! (If it doesn’t let me know, it’s a bug)

Do you need to do all that on your machine, all of it by hand? Luckily no, especially on GHA, where you’ve a bunch of actions already available. Let’s explore some of them in the next part of this post.

# Next

Let's look in the next post how to use all of the on GHA to build and test your projects on RISC-V. (Link incoming once posted.)
