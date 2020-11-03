---
title: Using .NET Bio with Windows Store apps
date: 2013-07-23T00:00:00.0000000
layout: post
categories: bio
tags: bio mvvm uwp dotnet
---

We recently published .NET Bio 1.1 ([download it here](https://github.com/dotnetbio/bio)) and included a version for Mono in the source tree which allows you to use C# and .NET Bio but run the applications on Linux or Mac machines when Windows just isn't available.  You can even use Xamarin Studio to build the apps which is quite cool.

Lately, I've spent a lot of time in Windows Store land building Windows Store apps - so I thought I'd try to port the Mono version over to the newer WinRT environment.  I had to modify the parser infrastructure and I currently only have FASTA running right now but I do have a version running as a WSA.  As a test I build a quick and dirty sequence visualizer - it prompts you for a FASTA file, loads it into a sequence array using .NET Bio and then renders out the sequences using a `RichTextBlock`:

![](/images/wsabio_screen1-1024x640.jpg "wsabio_screen1")

It is using a **SemanticView** control and when you "zoom out" of the visualization, it presents the same data as a bitmap:

![](/images/wsabio_screen2-1024x640.jpg "wsabio_screen2")

To accomplish this, it generates a **WritableBitmap** and colors in pixels based on nucleotide values.  Since this was not a sanctioned effort (I just wanted to play), I didn't upload the changed code to the bio project, but will instead include it here with this post so anyone else who wants to play with it can take the source code and do whatever they like.  Just a warning to other early adopters: I haven't tested a lot of this code, it's a quick and dirty effort!  Next, I plan to add in some support for alignments and then calling BLAST services.

[Here's the code](/samples/bio.wsa.zip)

Have fun!
