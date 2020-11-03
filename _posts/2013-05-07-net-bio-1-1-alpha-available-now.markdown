---
title: .NET Bio 1.1 alpha available now
date: 2013-05-07T00:00:00.0000000
layout: post
categories: bio
tags: bio
---

One of the open source projects I'm actively involved in is a bioinformatics library for .NET called, appropriately enough, .NET Bio. You can check it out on [GitHub](https://github.com/dotnetbio/bio). We have just put out the alpha version of the next release for community testing - this has several significant changes in it:

1. A new (more standard) implementation of the [Smith-Waterman alignment algorithm](http://en.wikipedia.org/wiki/Smith_waterman).
2. A new (more standard) implementation of the [Needleman-Wunsch alignment algorithm](http://en.wikipedia.org/wiki/Needleman%E2%80%93Wunsch_algorithm).
3. Several improvements to the Parallel [DeNovo assembler](http://en.wikipedia.org/wiki/Sequence_assembly) ([PADENA](https://mbf.codeplex.com/wikipage?title=PadeNA%3a%20A%20PARALLEL%20DE%20NOVO%20ASSEMBLER))
4. Changes to the NUCmer pairwise aligner to evaluate both the forward and reverse strands of the query sequence(s).
5. Better file compatibility with several standardized formats.
6. and lots of little performance tuning and bug fixes throughout.

Biology is one of those areas of science which needs high performance algorithms and computing power - we utilize the .NET Parallel Framework (PFx) all over the place to attempt to improve algorithmic performance and provide some great tools for analysis of biological sequences!
