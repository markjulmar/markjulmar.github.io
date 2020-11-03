---
title: Deferred bindings in WPF (ala Deferred Scrolling)
date: 2009-08-11T00:00:00.0000000
layout: post
categories: wpf
tags: wpf dotnet
---

One of the new features added to WPF 3.5 SP1 was the **ScrollViewer.IsDeferredScrollingEnabled** property which allows you to scroll through large amounts of data without taking the actual scrolling hit until the user stops moving the scroll thumb.  This is essential when you have complex visualizations, or if the data itself is virtualized and the load time is expensive.  You can get information on this feature here:

[http://msdn.microsoft.com/en-us/library/system.windows.controls.scrollviewer.isdeferredscrollingenabled.aspx](http://msdn.microsoft.com/en-us/library/system.windows.controls.scrollviewer.isdeferredscrollingenabled.aspx)

Recently, I was working on a project where I have a Slider that is controlling a grouping option.  When the user changes the slider value, the visuals are regrouped based on the position of the slider itself.  The problem I ran into was it's an expensive operation to do the grouping and if the user tries to quickly drag the slider I ended up doing a whole bunch of non-essential groupings of my data for no reason.  They are expensive enough that the actual drag of the slider was impacted by it - not to mention the CPU and rendering!  So, my initial response in these situations is to turn on asynchronous data binding - that's as simple as throwing the **Binding.IsAsync** flag.  This essentially causes the binding transfer to occur on a background thread and will often improve responsiveness in situations like this.

![](/images/TaxViewer1.jpg)

In this case, however, it didn't really help because I was still doing all the intermediate calculations.  What I really need is some way to _defer_ the binding transfer until the slider stops. There's no option for that however so I whipped up a simple class which allows me to bind two properties together and place a timer between them to indicate how long it should wait before transferring the **Source** to the **Target**.  I could have done it in the code - it's just a timer solution after all, but I wanted something reuseable.

Here's an example usage:

```xml
<StackPanel>
   <StackPanel.Resources>
      <DeferredBinder:DeferredBinding x:Key\="dbTest" Timeout="1" />
   </StackPanel.Resources>
   <TextBlock x:Name="tb" Text\="{Binding Source={StaticResource dbTest}, Path=Target}" FontSize="48pt" HorizontalAlignment="Center" />
   <Slider x:Name="slider" Margin="10" HorizontalAlignment="Center" Width="200" Minimum="0" Maximum="100" Value="{Binding Source={StaticResource dbTest}, Path=Source, Mode=OneWayToSource}" />
</StackPanel> 
```

In this case, the slider's value is transferred 1 second after the final update (i.e. it waits 1 second for the user to stop sliding).

Here's a sample project to try out:

[DeferredBindingSample.zip](/samples/DeferredBindingSample.zip)
