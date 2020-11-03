---
title: rCAT 2.0 is online, 3.0 is coming
date: 2010-01-28T00:00:00.0000000
layout: post
categories: bio
tags: bio
---

The main project I’ve been working on the past few months has been a rRNA sequencing application.  It’s a joint project involving Microsoft Research and the University of Texas in Austin.  The goal being to produce lightning fast visualizations (nucleotide, 2D and 3D) with very large (100,000 sequence) data sets on WPF.  It’s been a big learning experience for me in many ways because the traditional mechanisms for dealing with things in WPF just flat out fail when we load big datasets and start scrolling them around.  So, we’ve had to invent data virtualization schemes, our own UI virtualization for scrolling, several custom controls and a variety of other elements to pull it off so far.  rCAT 4.0 (coming in March) is even more ambitious with editing support for the sequences!

All that said, our current effort is now online with full source code – it’s interesting stuff to browse through even if you aren’t into molecular biology – check it out at [http://rcat.codeplex.com/](http://rcat.codeplex.com/)

Here’s a nice screen shot showing some of the elements – dockable tabs and sidebar items, birds-eye viewer, taxonomy viewer and custom colorization of nucleotide data.  And it, of course, is all MVVM.  Fun stuff!

![CATUI.Overview.Shrink](/images/CATUI.Overview.Shrink_thumb.png "CATUI.Overview.Shrink")
