---
title: MVVM Helpers v1.03
date: 2009-08-04T00:00:00.0000000
layout: post
categories: mvvm
tags: mvvm dotnet
---

I just put the latest version of the MVVM helpers online - [MVVMHelpers](https://github.com/markjulmar/mvvmhelpers).

There's a bunch of new stuff in it - check the release notes for the highlights.  There's a set of breaking changes in it as well, specifically I've moved several of the attached behaviors into the new Blend model.  Originally in my local version I did it to the JulMar.Wpf.Helpers.dll and got a dependency against System.Windows.Interactivity.dll.

I decided that for this release I didn't want to force that dependency so I created a secondary assembly JulMar.Wpf.Behaviors.dll which has all those behaviors in it.  The breaking change is I removed the original attached behaviors from the library (DoubleClickBehavior, NumericTextBehavior, MouseDragBehavior) in favor of using these new versions.  I did update the sample to show how they get used.

In a recent Essential WPF class, one of the students wanted an ObservableDictionary which we whipped up there - I cleaned up that implementation somewhat and added it into this library along with some unit tests for it.

Again, please check the readme included in the .zip file -- as always you are free to do whatever you like with this code.  If you do anything interesting or fun, let me know!
