---
title: 'Part 3: Shifting focus to the first available element in WPF'
date: 2008-09-12T00:00:00.0000000
layout: post
categories: wpf
tags: wpf dotnet
---

We've seen how to programmatically control focus and that's all great stuff, but one thing I like to do with WPF is see how much of the repetitive or UI-specific code I can move into the XAML and keep out of the code behind.  We can use the **FocusManager.FocusedElement** property to shift focus in XAML but it only works when the element exists in the main XAML file.  If you use UserControls it turns out that the approach doesn't work because that element is loaded separately and not available when the initial focus is being determined.

In my specific case, I have a wizard-style application which utilizes a TabControl to move between the pages.  The first page looks like this:

![](/images/TabPage_01.jpg)

The XAML layout for window1.xaml looks something like this:

```xml
<TabControl x:Name="Pages" SelectedIndex="0">  
   <FocusTest:Page1 />  
   <FocusTest:Page2 />  
   <FocusTest:FinalPage />  
</TabControl>
```

What I want is to have focus immediately positioned within the first text box but it turns out that you can't get WPF to do this directly from XAML - because the TextBox isn't a direct descendent of Window - it's part of the Page1 user control.  You can try putting the focus shift into the user control but it turns out that it won't work there either - it's got to be assigned to the focus scope which is the window.

So it might seem we are stuck with adding code behind logic (blech!) but all is not lost!  When I hit situations like this, I try to think about how to solve the problem generically so I can reuse my solution.  In this case, I decided to build a **MarkupExtension** to locate the first focusable element descendent.

If you are not familiar with markup extensions, they are a corner piece in the XAML extensibility story.  They allow for dynamic property assignment - where the value is determined at runtime vs. XAML compile time.  That's exactly what I need here - I want to find that TextBox at runtime and shift focus to it - just like I would have done in the code behind.

Creating a markup extension is trivial - you just extend the **MarkupExtension** base class and implement the **ProvideValue** method.  In this case, when provide value is called, I have to do several steps:

1. If the element we are bound to is not loaded yet, we need to wait for it.  We won't be able to find the child in the visual tree if the parent isn't yet loaded.
1. Once it *is* loaded, search the children and find the first control we can give focus to.  That means, the control is visible, focusable and enabled.
1. Assign logical focus to the new control in the closest focus scope parent, or just return the value if the property being assigned to is not **FocusManager.FocusedElement**.

I decided to try to make a generic markup extension that could be used outside my scenario so I added a little bit of code to see if we are assigning to **FocusManager.FocusedElement** and act differently in that specific case - otherwise the extension just returns the value.

With this new extension, I can now add a single line of code to each user control:

```xml
<UserControl x:Class="FocusTest.Page2"  
   xmlns:FocusTest="clr-namespace:FocusTest"  
   FocusManager.FocusedElement="{FocusTest:FirstFocusedElement}">
```

Now, when I run the application, focus is always assigned to the first control it can be assigned to.  It turns out that I can go one step better and **always** assign focus when the page becomes visible in my wizard - remember we have to wait for the control to finish loading to find the element.. this is done by hooking the **FrameworkElement.Loaded** event.  This event is raised each time the TabControl shifts to a new tab - so if we never unhook our handler, our focus management code is called each time the tab page becomes visible.  This might not be the required behavior so I added a simple property to the markup extension called OneTime to control that behavior.

There's a bit too much code to blog here, but if you want to try this out yourself, here's the test project.. feel free to use this however you like.  [FocusTest.zip (17.88 KB)](/samples/FocusTest.zip)

> **Update:**
> [Andrew Smith](http://agsmith.wordpress.com/category/wpf/) pointed out that the code would assign focus to a focus scope if it ran across one - which could happen if you have a toolbar or menu present in the UI.  I didn't in the sample (or in my production app) but I could certainly see that being a common reality.  I changed the above code to deliberately skip focus scopes and their children and keep going down the tree until it finds the first child.  Thanks Andrew!
