---
title: Adding NAS folders to Windows 7 libraries
date: 2009-02-24T00:00:00.0000000
layout: post
categories: personal
tags: personal
---

I use a lot of virtual machines - and it's always handy to store files onto the real machine through shared folders. With Windows 7 and the new library feature I thought it would be really cool to add the shares to the Documents Library but found that the underlying system must be running Indexing Server 4.0 -- not an option for my OSX based system!

Here's a nice little workaround that lets you add non-indexed folders though:

To add a non-indexed UNC as a library to Windows 7 Beta:  
  
1. Create a folder on your hard drive for shares. I used **c:\\users\\Mark\\Shares\\Documents**.  
1. Add the new folder to your library using Explorer.  
1. Delete the folder using a command prompt window.  
1. Use the `mklink` command to create a symbolic link with the Documents name to your share. In my case the proper command was:

    **mklink /D Documents \\.HostDocuments**

> *Note* that `mklink.exe` requires administrator access so start the command prompt by holding <kbd>SHIFT+CTRL</kbd> and clicking on it (or use the "Run As Administrator" option)

Voila! Explorer keeps the folder in the library even though it's not indexed.
