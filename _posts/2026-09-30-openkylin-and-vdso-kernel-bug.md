---
title: "An unrelocated _mcount call in the RISC-V vDSO"
layout: post
---

On an openKylin riscv64 VM running kernel `6.6.129-ur-cp100d-generic+`, programs using the glibc from Debian Trixie crash with a segmentation fault at startup. The fault is in the vDSO: `__vdso_riscv_hwprobe` calls `_mcount` through a PLT entry whose GOT slot is never relocated.

## Reproduction

The test program queries `RISCV_HWPROBE_KEY_IMA_EXT_0` through the raw `riscv_hwprobe` syscall, not through the vDSO:

```c
#define _GNU_SOURCE
#include <errno.h>
#include <stdint.h>
#include <stdio.h>
#include <string.h>
#include <unistd.h>
#include <sys/syscall.h>

#ifndef __NR_riscv_hwprobe
#define __NR_riscv_hwprobe 258
#endif

struct riscv_hwprobe {
    int64_t key;
    uint64_t value;
};

int main(void) {
    struct riscv_hwprobe p = {
        .key = 4,   /* RISCV_HWPROBE_KEY_IMA_EXT_0 */
        .value = 0
    };

    long r = syscall(__NR_riscv_hwprobe, &p, 1, 0, 0, 0);
    printf("return=%ld errno=%d (%s), key=%ld value=0x%lx\n",
           r, errno, strerror(errno), p.key, p.value);
    return 0;
}
```

I mounted the `debian:trixie` image with `ctr` and ran the program with the image's own dynamic loader and libraries:

```
ctr -n k8s.io images pull docker.io/library/debian:trixie
ctr -n k8s.io images mount docker.io/library/debian:trixie /mnt/debian-trixie

IMG=/mnt/debian-trixie
LOADER=$IMG/lib/riscv64-linux-gnu/ld-linux-riscv64-lp64d.so.1
"$LOADER" --library-path "$IMG/lib/riscv64-linux-gnu:$IMG/usr/lib/riscv64-linux-gnu:$IMG/lib:$IMG/usr/lib" ./hwprobe
Segmentation fault
```

The crash is deterministic. The same program runs correctly with the system's own loader and libraries. Since the program never calls the vDSO itself, the fault happens in the image's glibc during startup, before `main` runs.

## Locating the fault

With a breakpoint on `__vdso_riscv_hwprobe`, gdb stops on a call 30 bytes after the function entry:

```
(gdb) break __vdso_riscv_hwprobe
(gdb) r
Breakpoint 1, 0x00007ffff7fdab8c in __vdso_riscv_hwprobe ()
=> <__vdso_riscv_hwprobe+30>:  jal  0x7ffff7fdacf0
```

The call target has no symbol. Its first four instructions form a RISC-V PLT entry, which loads an address from a GOT slot and jumps to it:

```
0x7ffff7fdacf0:  auipc  t3,0x0
0x7ffff7fdacf4:  ld     t3,-1984(t3)
0x7ffff7fdacf8:  jalr   t1,t3
0x7ffff7fdacfc:  nop
```

The GOT slot contains `0xcd0`, and the jump faults on that address:

```
(gdb) x/g 0x7ffff7fdacf0-1984
0x7ffff7fda530: 0x0000000000000cd0
(gdb) si
Cannot access memory at address 0xcd0
```

## Why the GOT slot holds 0xcd0

The kernel maps the vDSO into each process without applying relocations, and glibc treats it as already relocated. A PLT entry in the vDSO therefore jumps to whatever value the linker wrote into the GOT at build time.

On RISC-V, the linker initialises each `.got.plt` slot with the address of the PLT header. In this vDSO, `.plt` starts at offset `0xcd0`, and the entry at `0xcf0` follows the 32-byte header. The slot still holds its link-time value, so it does not show which function the call was meant to reach.

## Inspecting the vDSO

The vDSO can be copied out of a running process through `/proc/self/mem` and inspected with the usual ELF tools. The script below reads it from its own process, which maps the same vDSO as every other process on the system. On success it only writes `/tmp/vdso.so`, so it logs the mapping it found and checks that the data is a complete ELF image before writing it:

```python
python3 - <<'EOF'
import sys

out = '/tmp/vdso.so'
start = end = None
with open('/proc/self/maps') as maps:
    for line in maps:
        if line.rstrip().endswith('[vdso]'):
            print(f'maps: {line.strip()}')
            start, end = (int(x, 16) for x in line.split()[0].split('-'))
            break

if start is None:
    sys.exit('error: no [vdso] mapping in /proc/self/maps')

size = end - start
print(f'vdso: 0x{start:x}-0x{end:x} ({size} bytes)')

with open('/proc/self/mem', 'rb') as mem:
    mem.seek(start)
    data = mem.read(size)

if len(data) != size:
    sys.exit(f'error: read {len(data)} of {size} bytes')
if data[:4] != b'\x7fELF':
    sys.exit(f'error: no ELF magic at 0x{start:x} (got {data[:4]!r})')

machine = int.from_bytes(data[18:20], 'little')
with open(out, 'wb') as f:
    f.write(data)
print(f'wrote {out}: {len(data)} bytes, ELF{64 if data[4] == 2 else 32}, '
      f'e_machine={machine} ({"RISC-V" if machine == 243 else "unexpected"})')
EOF
```

On the affected VM it prints:

```
maps: 7ffd42f8c000-7ffd42f8e000 r-xp 00000000 00:00 0                          [vdso]
vdso: 0x7ffd42f8c000-0x7ffd42f8e000 (8192 bytes)
wrote /tmp/vdso.so: 8192 bytes, ELF64, e_machine=243 (RISC-V)
```

```
$ readelf -Wr /tmp/vdso.so
Relocation section '.rela.dyn' at offset 0xd00 contains 1 entry:
    Offset             Info             Type               Symbol's Value  Symbol's Name + Addend
0000000000000530  0000000200000005 R_RISCV_JUMP_SLOT      0000000000000000 _mcount + 0

$ readelf -WS /tmp/vdso.so | grep plt
  [13] .plt              PROGBITS        0000000000000cd0 000cd0 000030 10  AX  0   0 16

$ readelf -Ws --dyn-syms /tmp/vdso.so | grep -w UND
     2: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT  UND _mcount
```

The only dynamic relocation is at offset `0x530`, the GOT slot read in gdb. It refers to `_mcount`, which the vDSO does not define. A correctly built vDSO has no dynamic relocations.

## Cause

GCC inserts a call to `_mcount` at the entry of every function compiled with `-pg`. The kernel builds with `-pg` when `CONFIG_FUNCTION_TRACER` is enabled and dynamic ftrace is not available. With dynamic ftrace, RISC-V kernels use `-fpatchable-function-entry` instead, which reserves NOPs at function entry rather than emitting a call.

The running kernel's configuration:

```
$ zcat /proc/config.gz | grep -E 'FUNCTION_TRACER|DYNAMIC_FTRACE'
CONFIG_HAVE_FUNCTION_TRACER=y
CONFIG_FUNCTION_TRACER=y
CONFIG_HAVE_DYNAMIC_FTRACE_WITH_DIRECT_CALLS=y
```

Function tracing is enabled, and neither `CONFIG_HAVE_DYNAMIC_FTRACE` nor `CONFIG_DYNAMIC_FTRACE` is present, so this kernel is compiled with `-pg`. I have not yet found out why the build did not select dynamic ftrace with GCC 14.2.

The vDSO Makefile removes the ftrace flags from the objects linked into the vDSO. The time functions contain no `_mcount` call, so the flags are removed for them. `hwprobe.o` is still compiled with them. Removing them for that object takes one more line in `arch/riscv/kernel/vdso/Makefile`:

```
CFLAGS_REMOVE_hwprobe.o = $(CC_FLAGS_FTRACE) $(CC_FLAGS_SCS)
```

With dynamic ftrace, the same build would leave NOPs at the start of `__vdso_riscv_hwprobe` instead of a call, and the problem would not show up.

## Scope and resolution

The fault only occurs when `__vdso_riscv_hwprobe` is called. Debian Trixie's glibc calls it at startup to select optimised routines for the CPU. The glibc shipped with openKylin does not appear to, which is why native programs on the VM run normally. Because the call happens during startup, before `main` and before any preloaded library runs, it cannot be worked around from userspace.

Two kernel build changes address it:

1. Remove the ftrace flags from `hwprobe.o` in the vDSO Makefile, as above.
2. Enable dynamic ftrace. Besides hiding this case, it avoids a call to `_mcount` at the entry of every kernel function when tracing is off.

Running `readelf -r` on `arch/riscv/kernel/vdso/vdso.so.dbg` after each build, and failing if it prints any relocation, catches this class of problem before it reaches a running system.
