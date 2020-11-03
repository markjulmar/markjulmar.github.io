---
title: Fixing SVCUTIL.EXE for Orcas Beta 2
date: 2007-07-30T00:00:00.0000000
layout: post
categories: dotnet
tags: dotnet
---

Microsoft apparently didn't re-sign all the tools in the SDK supplied with Beta 2 of Visual Studio 2008.  This is actually documented in the readme that UISpy has this issue, which of course, I didn't read..

**SVCUtil.exe** also has this problem and it causes any proxy-generation from VS.NET 2008 to fail (as well as killing the client tester process which is spawned for WCF testing).  You'll know you've hit this error when the program crashes with a `System.IO.FileException` and indicates that it has failed strong-name validation.

It's easy enough to fix ..

1. Open a command prompt with full admin rights
1. Change to the **C:\Program Files\Microsoft SDKs\Windows\V6.0a\bin** directory
1. Execute `SN -Vr svcutil.exe`

This turns off strong-name validation for **svcutil.exe** and allows it to function properly.
