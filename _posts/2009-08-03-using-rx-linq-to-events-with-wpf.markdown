---
title: Using Rx (Linq to Events) with WPF
date: 2009-08-03T00:00:00.0000000
layout: post
categories: wpf
tags: wpf dotnet
---

A recent series of blog entries at [http://themechanicalbride.blogspot.com/](http://themechanicalbride.blogspot.com/) introduced the Rx framework (System.Reactive.dll) which is an assembly used in the Silverlight toolkit for UI testing purposes.  It essentially provides a mechanism to do event driven programming through LINQ.  I'll refer you to the blog referenced above for all the gory details - frankly I'm still trying to wrap my mind around it!

Silverlight isn't my favorite technology however, I much prefer working in WPF and so I spent some time rebuilding Rx for the desktop CLR!  My original approach was to take my favorite exploration tool, reflector (available for free from [www.redgate.com](http://www.redgate.com)), and disassemble all the classes into C#, placing them into a project file.  I found however that Reflector choked on some of the more complicated structures - requiring me to go and hand-edit a bunch of the code.  I realized while I was doing this that there was a much easier way to convert a Silverlight assembly to a WPF assembly.

As you probably already know, Silverlight shares the same assembly format as the desktop CLR - there is no difference in the IL or structure of the assembly itself.  The difference is in the dependency on mscorlib and other references.  Specifically, when you add an assembly via VS2008, it looks at the **version** of the referenced mscorlib to determine whether it's a desktop CLR assembly or Silverlight assembly.

Here's the header of a Silverlight assembly examined through ILDASM:

```output
// Metadata version: v2.0.50727  
.assembly extern mscorlib  
{  
   .publickeytoken = (7C EC 85 D7 BE A7 79 8E ) // |.....y.  
   .ver 2:0:5:0  
}
```

Here is a desktop assembly:

```output
// Metadata version: v2.0.50727  
.assembly extern mscorlib  
{  
  .publickeytoken = (B7 7A 5C 56 19 34 E0 89 )                         // .zV.4..  
  .ver 2:0:0:0  
}  
```

Notice the version difference .. Silverlight uses 2.0.5.0 of mscorlib and the desktop CLR uses 2.0.0.0.  The public key is also different, but this is really just used for loading purposes.

Assuming the assembly doesn't use something Silverlight specific, we can modify the assembly references and allow the asembly to be used in the desktop CLR fairly easily. 

Here's how I did it for System.Reactive.dll:

```bash
ILDASM System.Reactive.dll /out:SR.il
```

change the assembly references in the resulting IL text file (for mscorlib, system and system.core)

```bash
ILASM SR.il /resource=sr.res /output=WPF/System.Reactive.dll
```

That gave me a desktop version of the assembly without modifying anything in the code itself - much easier than my original reflector route!

Next, I took a couple of the samples from the blog and ported them to WPF - specifically, I took the events sample listed here [http://themechanicalbride.blogspot.com/2009/07/developing-with-rx-part-1-extension.html](http://themechanicalbride.blogspot.com/2009/07/developing-with-rx-part-1-extension.html) and ported it to WPF to test it.  I made very few changes, primarily just switching `Application.Current.MainWindow` for `Application.Current.RootVisual`.  Here is the resulting project and switched assembly for anyone who is interested in it.

[ExtensionEvents.zip (44.62 KB)](/samples/ExtensionEvents.zip)

I think Rx is a fascinating piece of code, although it will take me a bit of time to realize just what I can do with it I suspect.  I'm looking forward to building more with this using WPF now that's for sure!
