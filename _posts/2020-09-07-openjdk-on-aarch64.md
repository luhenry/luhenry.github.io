---
title: "OpenJDK on AArch64"
layout: post
---

In response to recent developments around ARM64 (the [Apple Silicon](https://www.apple.com/newsroom/2020/06/apple-announces-mac-transition-to-apple-silicon/) announcement for example), the Java Engineering Group here at Microsoft decided to join in the effort to port the OpenJDK to ARM64 on Windows and macOS. I wanted to share some of the more interesting aspects of the work I'm involved in a series of blog posts. 

I'll document my journey to discover and resolve the differences between ARM64 and x86, on Linux, Windows, and macOS in these posts.
