---
title: Creating a legacy web service proxy in Visual Studio 2008
date: 2007-08-13T00:00:00.0000000
layout: post
categories: dotnet
tags: dotnet
---

If you right click on **References** in a VS2008 project you will find only two options now - "Add Reference" and "Add Service Reference".  The first is for local assemblies, and the second generates WCF-compatible proxies through SVCUTIL.EXE.  This happens to be one of the most obviously changed things in Orcas - the dialog presented is a more professional version than what was supplied with the original WCF CTP for VS.2005:

![](/images/add-service-reference.jpg)

The "Discover" button above can locate IIS-hosted web services (not Self Hosted however) and it now shows operations directly which is pretty cool.  The best part is the "Advanced" button though.  Clicking that gives you:

![](/images/ScreenShot001.jpg)

Here you can control all the details of `SVCUTIL.EXE` -- how to reuse types, whether to generate public/private types, whether to generate lower-level message contracts (giving complete control over the SOAP structure), and whether to generate asynchronous methods (ala the original web services references in .NET 2.0).

The bottom section - "Compatibility" allows you to generate a web service proxy using that original WS technology.  Clicking this will yield the traditional dialog you are probably used to:

![](/images/ScreenShot002.jpg)

So, if you were confused by the lack of a "Add Web Reference" in Orcas, fear no more - it's still there just a little harder to get to!

You might be asking - "Why would I want to generate one of those?  Why not just use WCF?"  Well, we have to keep in mind that not all our clients want to move to WCF just yet (as much as we want them to, there is a cost in deployment), and second, WCF isn't supported on pre-XP platforms.  There are still a lot of Windows 2000 systems out there!  For those clients, we have no choice but to use legacy web service proxies or [shared proxy servers](http://fineproxy.de/blog/) from a reputable online seller.
