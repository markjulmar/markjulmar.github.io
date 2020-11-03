---
title: It's been such a long time.. [WPF + Windows 7]
date: 2009-01-29T00:00:00.0000000
layout: post
categories: wpf
tags: wpf dotnet
---

> **Update**
> Microsoft has released the API code pack including Windows 7 support -- get it here: [http://code.msdn.microsoft.com/WindowsAPICodePack](http://code.msdn.microsoft.com/WindowsAPICodePack)  
  
Well, it's been a while since I posted anything, I'm sorry!

I've been busy working with Windows 7 and Microsoft Surface touch-computing. To that end, I needed to get access to some of the new Windows 7 APIs in managed code ... which isn't supported yet (but is coming).  
  
So I built an interop library to access:

- **Scenic Ribbon (native Win32 ribbon) in Windows Forms**
- **Native WM\_TOUCH and WM\_GESTURE messages**
- **Sensor API**
- **Shell APIs (jump lists, thumbnail buttons, libraries)**

> **Note:**
>
> I make no guarantee that everything is correct (it's hard to do that on a shifting beta with minimal docs, but in addition I'm not sure I've gotten 100% coverage with everything anyway).

Here's the interop library with source code:  
  
[Windows 7 Beta 1 Interop Library for .NET 2.0](/samples/Win7Interop.zip)
  
I'll be posting more details and samples a bit later, here's a couple to get you started.  
  
Here's a simple example of using the Scenic Ribbon and native touch support to create a (very) simple Windows Forms finger paint program:  
  
![touch_img.jpg](/images/touch_img.jpg)  
  
[Multi Touch example with Windows Forms](/samples/touch_sample.zip)  
  
Here's a simple example of using the gestures and library support in a WPF application. It grabs all the directories in your Pictures Library and then shows you each picture and lets you use the swipe gesture to move between then, pinch to scale and of course, rotate.  
  
![gesture_img.jpg](/images/gesture_img.jpg)  
  
[Gesture example with WPF](/samples/gesture_sample.zip)  
  
Both of these samples work with the HP Touchmate (and multi-touch drivers) and Windows 7 Beta 1.

Have fun!
