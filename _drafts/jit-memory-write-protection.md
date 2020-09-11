---
title: "JIT Memory Write Protection"
layout: post
---

_This is an installment in a [series of posts]({% post_url 2020-09-07-openjdk-on-aarch64 %}) that will highlight discoveries I am making as I add support for Windows-AArch64 and macOS-AArch64 to the OpenJDK._

Apple has, over the years, taken measures to improve the security of macOS applications. Lately, they have introduced [Hardened Runtime](https://developer.apple.com/documentation/security/hardened_runtime), a set of technologies to "Manage security protections and resource access for your macOS apps." 



- Hardened Runtimes
- https://developer.apple.com/documentation/apple_silicon/porting_just-in-time_compilers_to_apple_silicon
- JitWriteUnprotect
- signal handler fallback
