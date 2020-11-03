---
title: ATAPI.NET updated for VB.NET
date: 2006-04-20T00:00:00.0000000
layout: post
categories: tapi
tags: tapi
---

Minor, but breaking update for ATAPI.NET (version number has changed).  It was pointed out to me that the assembly wasn't very VB.NET friendly in that it didn't show any events at the `TapiManager` level and you couldn't use the UI to hook it all up.  That's fixed and all the line-level events are also exposed at the `TapiManager` level for those who want to use VB.NET with the 2.0 wrapper. Source code is [here](https://github.com/markjulmar/atapi.net).
