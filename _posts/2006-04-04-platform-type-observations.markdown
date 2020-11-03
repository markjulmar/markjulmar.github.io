---
title: Platform type observations..
date: 2006-04-04T00:00:00.0000000
layout: post
categories: dotnet
tags: dotnet
---

Recently someone informed me that the TAPI wrappers (ITAPI3 and ATAPI) do not appear to function properly on the Win64 platform.  It throws an exception that says:

```output
"Could not load file or assembly 'ITapi3, Version=1.0.0.3, Culture=neutral, PublicKeyToken=36377d9f6f1f4883' or one of its dependencies. An attempt was made to load a program with an incorrect format."
```

Now, as most know, .NET code compiles to an Intermediate Language (IL) which is a bytecode that is translated to the appropriate processor-specific instructions at runtime by the just-in-time (JIT) compiler.  The type of code that is generated is determined by the version of the CLR that is loaded.

When a .NET application starts, **mscoree.dll** is responsible for determining the proper version of the CLR to start for the process.  This is done by looking at the application manifest to see which version of .NET the application was compiled for (1.0, 1.1, or 2.0) and mscoree will suck in the appropriate assemblies and DLLs for that specific version, or in some cases, upgrade it silently to a newer version.

On Win64, the picture is a little more complex because we have to consider the platform as well.  There are actually two 2.x versions of .NET on Win64 - a 32-bit version (which is the same as the one running on Win32) and a 64-bit version which has been optimized for the 64-bit platform and generates 64-bit JITted code.  When an assembly starts on a Win64 platform, the **mscoree.dll** will look at not just the version, but also the platform flag which is coded into the manifest.  We can see this flag using ILDASM:

![](/images/ildasm1.JPG)

The **.corflags** above tells the loader that this particular assembly **requires** the 32-bit CLR, in other words, that we have a dependency on a 32-bit resource such as a COM object or platform DLL.  By default, the flag will be set to 0x0000001 (ILONLY) indicating no dependency (VS.NET refers to this as "AnyCpu" in the platform flag settings) and, on a Win64 machine, the assembly will be loaded under the 64-bit CLR.  With it set as above, on a Win64 machine it would be loaded into the 32-bit CLR.  For those who are interested in this aspect, there is a tool in the SDK (CORFLAGS.EXE) that will let you manipulate this flag and force an ILONLY assembly to be loaded as 32-bit.

VS.NET allows you to change the platform type on an assembly-by-assembly basis through the project settings, Build Tab:

![](/images/platformtype.JPG)

When loading, this platform CLR determination appears to be, as with the CLR versioning, based solely on the assembly that starts the process.  The loader doesn't scan the dependencies and determine that one of the required assemblies is marked as '"x86" and will simply fail the process when that assembly eventually gets loaded.

So, if I have an assembly that requires 32-bit execution (such as ATAPI.NET or ITAPI3 that depend on the 32-bit TAPI subsystem and COM objects), but my starting assembly is marked as "AnyCpu", then the loader will start it under Win64 as a 64-bit process and when it tries to initialize ATAPI.NET or ITAPI3, it will fail the AppDomain with an exception.  Marking ATAPI.NET and ITAPI3 as "x86" (which they are by the way) won't help in this case -- you **must** mark your application as 32-bit.

This really is a bummer and, while I'm not surprise the CLR loader doesn't do it, I am surprised that VS.NET doesn't force the .EXE to be "x86" when it sees a dependency that requires it.

The wrappers _do_ function under Win64, but _only_ if the starting application has the platform target marked as "x86".
