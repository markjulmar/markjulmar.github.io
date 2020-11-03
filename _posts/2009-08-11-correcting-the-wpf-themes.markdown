---
title: Correcting the WPF Themes
date: 2009-08-11T00:00:00.0000000
layout: post
categories: wpf
tags: wpf dotnet
---

I'm a big fan of themes in WPF -- and I think it's great that Microsoft has released a whole set of themes for Silverlight and WPF at [www.codeplex.com/wpf](http://www.codeplex.com/wpf)

However, in using some of those themes, I've run into an annoying bug in the RadioButton template - specifically, the checked state doesn't always show up initially.  Looking at the template, it turns out to be an easy fix.  The "CheckIcon" is set to an opacity of zero initially (so it's not shown) and then a trigger is used to apply an animation to change the value.  Unfortunately, it looks like the animation is switched - it applies when the checkbox is **UNCHECKED** vs. **CHECKED**.  So, two ways to fix it -- either change the initial opacity to "1" for the "CheckIcon" element in the control template, or go to the triggers in the control template for the RadioButton and swap the states so it looks like this:

```xml
<Trigger Property="IsChecked" Value="false" />
<Trigger Property="IsChecked" Value="True">
   <Trigger.EnterActions>
      <BeginStoryboard x:Name="CheckedOn\_BeginStoryboard" Storyboard="{StaticResource CheckedOn}"/>
   </Trigger.EnterActions>
   <Trigger.ExitActions>
      <BeginStoryboard x:Name="CheckedOff\_BeginStoryboard" Storyboard="{StaticResource CheckedOff}"/>
   </Trigger.ExitActions>  
</Trigger\>
```
