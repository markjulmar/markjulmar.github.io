---
title: 'Blend Behaviors in Styles: DragPositionBehavior'
date: 2010-02-05T00:00:00.0000000
layout: post
categories: mvvm
tags: mvvm dotnet
---

In the last post, I referred to the `DragPositionBehavior` in the [JulMar MVVM library](https://github.com/markjulmar/mvvmhelpers/).  This behavior allows any `UIElement` to be dragged and repositioned using the mouse without requiring any code logic on your part.  It’s easy to apply – using the traditional Blend syntax (easiest done by dragging the behavior onto an element):

```xml
<Interactivity:Interaction.Behaviors\>
    <julmar:DragPositionBehavior />
</Interactivity:Interaction.Behaviors\>
```
  
This is the simplest way to use this, however it doesn’t work when the behavior is to be applied with a `Style` – in my example in the prior post, we need to drag around the `ListBoxItem` container, not just the content.  So I showed an alternative syntax to apply the same behavior:

```xml
<ListBox.ItemContainerStyle\>
    <Style TargetType\="ListBoxItem">
        <Setter Property\="julmar:DragPositionBehavior.IsEnabled" Value\="True" />
        ...
```
  
Here, the behavior is being applied by setting the `IsEnabled` attached property onto the `ListBoxItem`.  Here’s the implementation for that:

```csharp
public static DependencyProperty IsEnabledProperty = DependencyProperty.RegisterAttached("IsEnabled", typeof(bool),  typeof(DragPositionBehavior), new FrameworkPropertyMetadata(false, OnIsEnabledChanged));

public static bool GetIsEnabled(DependencyObject uie)
{
    return (bool)uie.GetValue(IsEnabledProperty);
}

public static void SetIsEnabled(DependencyObject uie, bool value)
{
    uie.SetValue(IsEnabledProperty, value);
}
```

This is a boilerplate example of an attached property – nothing to see here.  The magic happens in the `PropertyChange` callback:

```csharp
private static void OnIsEnabledChanged(DependencyObject dpo, DependencyPropertyChangedEventArgs e)
{
    UIElement uie = dpo as UIElement;
    if (uie != null)
    {
        var behColl = Interaction.GetBehaviors(uie);
        var existingBehavior = behColl.FirstOrDefault(b => b.GetType() ==  typeof(DragPositionBehavior)) as DragPositionBehavior;
        if ((bool)e.NewValue == false && existingBehavior != null)
        {
            behColl.Remove(existingBehavior);
        }
        else if ((bool)e.NewValue == true && existingBehavior == null)
        {
            behColl.Add(new DragPositionBehavior());
        }
    }
}
```
  
Let’s break this down a little. When we have the `IsEnabled` property set onto an element, first we verify it’s a `UIElement` (we cannot drag it on anything that isn’t because the mouse events are defined at this level). Next, we check the behaviors collection on that element – if an existing behavior is there and we are setting the property to “false”, then we remove the existing behavior from the collection. If there is no behavior, and we are setting the property to **true**, then we add a new `DragPositionBehavior` instance to the collection here. This, in effect, is exactly what the first block of XAML is doing, and it’s what we do as well here in code – so the Style setter works as expected.

This isn’t really a trick – I’m sure others have thought of it as well, but it’s a useful thing to add into your behaviors so they can be universally used, both by Blend as well as by developers directly wanting to apply it in places where Blend cannot today.
