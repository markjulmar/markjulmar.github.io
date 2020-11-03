---
title: 'Part 2: Changing WPF focus in code'
date: 2008-09-04T00:00:00.0000000
layout: post
categories: wpf
tags: wpf dotnet
---

In the last post, I wrote about how focus is generally managed in WPF - we have focus scopes to track a single element within that scope for logical focus, and then one of those elements is given physical, or keyboard focus.

Now, let's talk a little about how you can influence that programmatically.  First, you can always determine which element has logical focus in your application through the **FocusManager.GetFocusedElement** method -- pass it the window in question and it will return which element has logical focus in that window.  Remember that logical focus != keyboard focus at all times -- toolbars and menus track their own focus so if you are currently interacting with a menu then the menu has physical focus.  But in general, the following code will tell you which element WPF thinks has focus in the window:

```csharp
IInputElement focusedElement = FocusManager.GetFocusedElement(thisWindow);
```

To determine whether this element has keyboard focus, we can check the IsKeyboardFocused property - if it's set to true, then that element currently has the keyboard focus (as well as being the logical focus for that focus scope).

Keyboard focus is most often set through runtime activity - the user clicks on an element, or uses the TAB key to move around the UI.  You can also set it programatically a couple of ways.  First, there is a **Keyboard** class in WPF which exposes several methods and properties.  There is a **Keyboard.FocusedElement** read-only property which returns the current keyboard focused element, and there is a **Keyboard.Focus** method which attempts to change keyboard focus.  It returns the element that now has focus - so you can check to see if your request was fulfilled or ignored.  So, for example, you can change focus during your application initialization:

```csharp
void OnLoaded(object sender, RoutedEventArgs e)
{
    Keyboard.Focus(firstTextBox);
}
```

Notice that we uses the **Loaded** event - this is because no focus requests will be accepted prior to the element being initialized and loaded.  That's the first place in the application where you can make focus changes.

When would setting focus fail?  Well, it can fail for a lot of reasons, but the most common are:

1. The element has `Focusable` = **false**
1. The element has `Enabled` = **false**
1. The element has `IsVisible` = **false**
1. The element has not been loaded yet
1. The currently focused element will not release focus.

That final one is important, changing focus involves potentially taking it away from an existing element - they receive a **PreviewLostKeyboardFocus** and **LostKeyboardFocus** event.  If they handle the preview event, focus will not change.

You can also manipulate focus through programmatic keyboard navigation - simulating the user pressing TAB to cycle through the focusable elements.  This is controlled through the **KeyboardNavigation** class which is used when the user presses a key that changes focus (TAB, SHIFT+TAB, Up, Down, etc.).

Controls can set a **TabIndex** property assignment which determines the tabbing order.  The default is to tab through them in order of declaration.  You can also use the **KeyboardNavigation.TabIndex** attached property which works for any element - not just controls.

To control navigation, the **KeyboardNavigation** class has an attached property **TabNavigation** allowing you to change how navigation occurs within a container.  You can set it to:

1. **Continue** - each focusable element receives focus and the container is exited when the edge is reached.
1. **Cycle** - focus does not leave the container but wraps around the edges
1. **Once** - the container itself is treated as a single focusable element where only the first child receives focus
1. **Local** - uses TabIndex locally within the container - independent of any outside elements.
1. **Contained** - focus stays in the container but does not wrap (stays at edges when top/bottom are reached)
1. **None** - no keyboard navigation allowed in the container

The default is **Continue**, but you can set the attached property on any element to change it for that element and any children. To see this in action, paste the following into your XAML editor of choice and change the `ComboBox` while tabbing through the `TextBlock` elements.

```xml
<Window xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
    xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
    xmlns:sys="clr-namespace:System;assembly=mscorlib"
    Title="Simple Focus">

    <Window.Resources>
        <Style TargetType="TextBox">
            <Setter Property="Margin" Value="10" />
            <Setter Property="Width" Value="100" />
        </Style>
    </Window.Resources>

    <StackPanel>
        <ComboBox x:Name="tabStyles" SelectedIndex="0" Focusable="False">
            <sys:String>None</sys:String>
            <sys:String>Continue</sys:String>
            <sys:String>Cycle</sys:String>
            <sys:String>Once</sys:String>
            <sys:String>Local</sys:String>
            <sys:String>Contained</sys:String>
        </ComboBox>
        <TextBox TabIndex="1" />
        <TextBox TabIndex="2" />
        <StackPanel KeyboardNavigation.TabNavigation="{Binding ElementName=tabStyles,Path=SelectedItem}">
            <TextBox TabIndex="1" />
            <TextBox TabIndex="2" />
        </StackPanel>
    </StackPanel>
</Window>
```

You can have the system ignore specific elements (but still allow them to have focus) by setting the **KeyboardNavigation.IsTabStop="false"** attached property.  This will cause keyboard navigation to "jump" over the control as if it were not present.

Three methods are exposed by `UIElement` and `FrameworkElement to` programmatically shift focus: **Focus**, **MoveFocus** and **PredictFocus**.

To force focus to a specific element, you can call **Focus** on it.  For example, above we set the keyboard focus by calling Keyboard.Focus(), but the same effect can be achieved like this:

```csharp
void OnLoaded(object sender, RoutedEventArgs e)
{
    firstTextBox.Focus();
}
```

This method attempts to set focus using `Keyboard.Focus()`.  If that fails, but the element is `Focusable` and enabled, it finds the focus scope for the element and sets logical focus there (so that keyboard focus will eventually end up on the control).

**FrameworkElement.MoveFocus** is used to change the keyboard focus in the application using the same algorithm as the TAB traversal.  You pass in the direction (specified through a TraversalRequest object) and the method returns true/false to indicate success.   Under the covers it actually uses the KeyboardNavigation class, but it's an easy way to push focus around the window:

```csharp
firstTextBox.MoveFocus(new TraversalRequest(FocusNavigationDirection.Next));
```

**PredictFocus** works the same way, but instead of actually shifting focus, it returns what _would_ be the focused item if you were to execute MoveFocus.

So, up to this point, we've seen a lot of code to change focus.  However, the most common request is to set initial focus to a specific control - remember that WPF doesn't do that by default.  You can do it in code, just like the above example where we use the Loaded event.  Or, it turns out you can do it in XAML too.  The key to remember is that the FocusedElement of the main focus scope (the Window) is the one that will get initial focus.  That is (by default) null, but you can set it in XAML using the attached property syntax.  Using the above XAML example, we can supply a name for one of the TextBox controls and then a little data binding magic to set that onto the Window:

```xml
<Window xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
    xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
    xmlns:sys="clr-namespace:System;assembly=mscorlib"
    Title="Simple Focus" 
    FocusManager.FocusedElement="{Binding ElementName=tb2}">
   ...
    <TextBox TabIndex="1" />
    <TextBox x:Name="tb2" TabIndex="2" />
     ... 
</Window>
```

Now when you run the application, focus is placed into _the_ second text box in the window.  This technique works great as long as the element you want to assign focus to is declared here in the same XAML file.  However, a popular way to develop WPF applications is to separate out chunks of UI into separate UserControls.  When you do that, the above trick fails -- even if you put the `FocusManager.FocusedElement` binding into the UserControl!

How we solve that is what we'll look at in the next post!  Stay tuned...
