---
title: 'Focusing on WPF:  How to deal with Focus() in the WPF world'
date: 2008-09-01T00:00:00.0000000
layout: post
categories: wpf
tags: wpf dotnet
---

Recently I was involved in a project where we needed to build a multi-step input application where each step showed progress and you could proceed forward and backward through the pages. I looked at the Navigation support in WPF (which is nice) but ultimately decided to model it around a styled **TabControl** – each page being a tab and the progress noted through the custom **TabItem** visuals across the top. It all works great and looks fabulous.

![](/images/navpage_ss.jpg.jpg)

One of the bugs that was logged against the application and got assigned to me was related to focus – the initial keyboard focus was not being placed into the first control, instead it appeared that nothing had focus which is totally accurate. Those that are new to WPF might be surprised that it does not assign initial focus to any particular child control – you must deliberately click or tab into a control to give it focus. You can, of course, also assign focus programmatically but it turns out to not be as easy as you’d think. In fact, I encountered several nasty gotchas trying to get it to work exactly the way I wanted it to.

My frustrations with focus and how WPF manages it have resulted in this set of blog posts

- [Part 1: Focus Basics](../part-1-its-basically-focus/)
- [Part 2: Setting focus in code and XAML](../part-2-changing-wpf-focus-in-code/)
- [Part 3: Getting a little smarter – setting focus to the first focusable control](../part-3-shifting-focus-to-the-first-available-element-in-wpf/)

