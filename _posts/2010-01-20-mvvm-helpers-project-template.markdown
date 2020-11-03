---
title: MVVM Helpers Project Template
date: 2010-01-20T00:00:00.0000000
layout: post
categories: mvvm
tags: mvvm dotnet
---

To start off this new series, I have created a MVVM Helpers project template - this is a Visual Studio 2008 template which will create the starter project I will be using as an example. It shows off the primary usage of the MVVM library and has directories and references to required assemblies already established so it's a nice starting point for any MVVM WPF app using the helpers.  
  
Download it from [here](/samples/WpfMvvmApplication.zip) and copy the zip file into your template directory which will be something like:  
  
**"C\:Users\{username}\Documents\Visual Studio {version}\Templates\ProjectTemplates\Visual C#\Windows"**

Here's what my directory looks like:  
  
![templatedir.jpg](/images/templatedir.jpg)

With that in place you should be able to fire up Visual Studio 2008 and create a sample MVVM project:

![project_screen.jpg](/images/project_screen.jpg)

Let it generate a project and the build it -- running the program will give you a simple file explorer using the MVVM design pattern and WPF:

![sample-run1.jpg](/images/sample-run1.jpg)

In tomorrow's post, we'll break the sample down and see how I built it. See you then!
