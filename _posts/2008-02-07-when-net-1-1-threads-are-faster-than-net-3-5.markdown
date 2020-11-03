---
title: When .NET 1.1 threads are faster than .NET 3.5!
date: 2008-02-07T00:00:00.0000000
layout: post
categories: dotnet
tags: dotnet
---

Today at the GNET [Michael Kennedy](http://www.develop.com/us/technology/bio.aspx?id=107), [Jason Whittington](http://www.develop.com/us/technology/bio.aspx?id=52) and I discovered a pretty interesting change in .NET 2.0 SP1 (included in .NET 3.5).

We knew they increased the thread pool limit from 25 threads/cpu to 250 threads/cpu, but we also found they changed the heuristics for creating threads to avoid creating a bunch of threads in a short period of time. In previous versions of .NET, threads were allocated at a rate of 1 every 500ms assuming there were queued items. This is documented behavior [in MSDN](http://msdn2.microsoft.com/en-us/library/0ka9477y.aspx).

In .NET 3.5, it appears to have been changed to a sliding scale where they slow down thread creation as more demand is placed on the pool. This is a change even from the beta and must have been slipped in at the last minute.

[Michael documented the whole change on his blog here](http://www.michaelckennedy.net/blog/PermaLink,guid,55a9b21e-ae85-4c24-a0b6-63dff4a6b491.aspx).. recommended reading.
