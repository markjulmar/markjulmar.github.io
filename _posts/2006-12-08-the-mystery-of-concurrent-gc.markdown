---
title: The Mystery of Concurrent GC
date: 2006-12-08T00:00:00.0000000
layout: post
categories: dotnet
tags: dotnet
---

There's been a discussion going on within **DevelopMentor** for a couple weeks regarding concurrent GC and when it really applies.

The idea behind the concurrent collector is to do as much of the GC while the UI thread continues to process UI stuff and then only interrupt the application threads when memory is being shuffled around and fixups are occurring. This provides for more responsive UI applications at the expense of slower collections and a higher memory utilization.

The instructors who are involved with .NET were discussing this because of a discrepancy in various reliable sources - First, Maoni Stephens, the ultimate authority on all things .NET GC, stated in her blog entries that concurrent GC was the default behavior for the workstation version of GC (see [http://blogs.msdn.com/maoni](http://blogs.msdn.com/maoni/) for more information on this).

This is well known, and now well documented in various places. However, Jeff Richter seems to say in his book "_CLR via C#_" that concurrent collections occur only on multiprocessor machines. Several other authors back this up (notably Stephen Pratschner in "_Customizing the .NET Framework Common Language Runtime_" and Joe Duffy in his "_Professional .NET Framework 2.0_" book; both are excellent btw).

So, we had some differing opinions from people in the "know". Our [Effective .NET](http://www.develop.com/training/course.aspx?id=344) and [Essential .NET](http://www.develop.com/training/course.aspx?id=267) courses were written around Maoni's blogs and so the diagram we presented showed concurrent collections even on single-processor machines since she didn't explicitly say it required multiple processors to turn it on. One of the guys noticed this and said _"Wait! That's wrong!"_

We tried to actually see the concurrent collection in action but it turns out that it's actually quite difficult to get this to happen because concurrent collections only occur on **Gen2**, which for a well-written application shouldn't occur that often. To add to the complexity, the CLR attempts to optimize this behavior - just because it _can_ do the GC concurrently, it might not.  Finally, the thread which is used for this collection is actually created and destroy as necessary so it doesn't exist most of the time.

So, to settle the argument, Jason Whittington and I spent some time spelunking into the CLR this past week to see if we could spot the concurrent collection in action. With a little WinDBG and some symbols, I think we've put the question to rest once and for all (at least for us) :-).  If you scan the GC symbols in **mscorwks.dll**, you'll find a treasure trove of information; one of the things that caught my eye was this:

```output
0:000> x mscorwks!WKS::gcheap::*
79f8c5dd mscorwks!WKS::GCHeap::Relocate = <no type information>
7a0d09d9 mscorwks!WKS::GCHeap::ValidateObjectMember = <no type information>
7a088e2a mscorwks!WKS::GCHeap::IsConcurrentGCInProgress = <no type information>
```

Disassembling that method showed me that it checks a flag in memory --

```output
0:000> u mscorwks!WKS::GCHeap::IsConcurrentGCInProgress 
mscorwks!WKS::GCHeap::IsConcurrentGCInProgress:
7a088e2a a1701b387a      mov     eax,dword ptr [**mscorwks!WKS::gc_heap::settings+0x10** (7a381b70)]
7a088e2f c3              ret 
```

I tried putting a breakpoint at the end of the GC process and dumping that flag out when it was a Gen2 collection (the breakpoint itself stolen shamelessly from [Maoni's MSDN article in November 2006](http://msdn.microsoft.com/msdnmag/issues/06/11/CLRInsideOut/default.aspx)):

```output
0:000> bp mscorwks!WKS::GCHeap::RestartEE "j (dwo(mscorwks!WKS::GCHeap::GcCondemnedGeneration)==2)   
'dd mscorwks!WKS::gc_heap::settings+0x10 L1';'g'"
```

But I didn't get much success seeing it set to "1" - I suspect that it requires a more complex application than I was using to cause the CLR to decide a concurrent collection is worth the effort. So I decided this was a dead end and started looking to see where the GC process determines that concurrent collections are allowed at all. I figured **GCHeap::Initialize** sounded like a good spot --

```output
0:000> u mscorwks!WKS::GCHeap::Initialize
mscorwks!WKS::GCHeap::Initialize:
79ed6924 a14412387a      mov     eax,dword ptr [mscorwks!g_pConfig (7a381244)]
79ed6929 8b4058          mov     eax,dword ptr [eax+58h]
.....
79ed697b 51              push    ecx
79ed697c e823040000      call    mscorwks!WKS::gc_heap::initialize_gc (79ed6da4)
79ed6981 85c0            test    eax,eax 
```

Going into that function yielded what I was looking for --

```output
0:000> u mscorwks!WKS::gc_heap::initialize_gc
mscorwks!WKS::gc_heap::initialize_gc:
79ed6da4 56              push    esi
79ed6da5 e80cf1ffff      call    mscorwks!WKS::write_watch_api_supported (79ed5eb6)
79ed6daa 33f6            xor     esi,esi
...
79ed6dc6 c70564ce387a01000000 mov dword ptr [**mscorwks!WKS::gc_heap::gc_can_use_concurrent** (7a38ce64)],1 
```

Looking at that particular memory location, I tried various configurations to make sure I had the right spot. First, I did a simple console app with no config file and dumped out the location on a multi-processor box:

```output
0:000> dd mscorwks!WKS::gc_heap::gc_can_use_concurrent L1
7a38ce64  00000001
```

Next, I set the application up in server mode (also on a multi-processor box):

```xml
<xml version="1.0" encoding="utf-8" ?>
<configuration>
  <runtime>
    <gcServer enabled="true" />
  </runtime>
</configuration>
```

```output
0:000> dd mscorwks!WKS::gc_heap::gc_can_use_concurrent L1
7a38ce64  00000000
```

Showing that it was now off.. next I tried a VM session which emulates a single processor:

```output
0:000> dd mscorwks!WKS::gc_heap::gc_can_use_concurrent L1
7a38ce64  00000001
```

It appears that the CLR **is** capable of doing concurrent collections on a single processor! My final test was to try turning it off altogether via a configuration switch:

```xml
<xml version="1.0" encoding="utf-8" ?>
<configuration>
  <runtime>
    <gcConcurrent enabled="false" />
  </runtime>
</configuration>
```

```output
0:000> dd mscorwks!WKS::gc_heap::gc_can_use_concurrent L1
7a38ce64  00000000
```
  
As a final note on this, Jason asked Maoni directly and her response was that concurrent GC does **not** depend on the number of processors - just as we are seeing above. So, apparently several primary sources are actually incorrect on this behavior. It just goes to show how complex the overall GC process really is -- simple in concept, but boy there are a lot of moving parts!
