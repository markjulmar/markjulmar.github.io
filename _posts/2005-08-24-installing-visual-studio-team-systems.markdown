---
title: Installing Visual Studio Team Systems
date: 2005-08-24T00:00:00.0000000
layout: post
categories: dotnet
tags: dotnet
---

So I spent the better part of a day getting a full Visual Studio Team Systems installation running under VMWare.  In order to get it all to work together on a single server, you have to have a bunch of components: Active Directory, SQL Server 2005, SQL Server Reporting Services, SharePoint, Team Foundation Server, Office and finally VSTS itself.  The first problem I ran into was just plain ignorance about SharePoint.  I started with a prebuilt W2K3 server image that already had IIS setup.  I then turned it into a domain server by installing Active Directory and then followed all the instructions.  
  
I noticed that SharePoint didn't run properly and it appeared to be security related.  I solved the issue by manually changing the security settings on the website, thought to myself "well that's odd..." and moved on.  When I tried to create my first VSTS project in the VM, it blew up saying SharePoint wasn't working properly.  Ah.. ok.  So, a bit of reading on the SharePoint admin guide and I run into the statement .. "You must install IIS after installing Active Directory".  Sigh. So, I blow away the image and start over.  Nothing more fun that reinstalling a bunch of DVD images right?  
  
The next run went a bit more smoothly, SharePoint appeared to work without me changing anything, but I again had the same issue creating a project.  I check the event log and find that Reporting Services won't start properly.  Weird.. it started before.  I find out that it's because Reporting Services is configured for .NET 2.0 and SharePoint is configured for .NET 1.1 and they are trying to run in the same host.  Obviously, that doesn't work - we can only start one version of the CLR at a time.  SharePoint doesn't appear to run properly under 2.0 (or at least I couldn't get it to run correctly and I didn't want to RTFM).  Instead, I found that separating the two applications out to separate application pools did the trick and I'm now off and running .. albeit slowly under a VM.

Ahhh.. let the modeling begin.
