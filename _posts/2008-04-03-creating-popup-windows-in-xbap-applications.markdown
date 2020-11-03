---
title: Creating popup windows in XBAP applications
date: 2008-04-03T00:00:00.0000000
layout: post
categories: wpf
tags: wpf dotnet
---

A colleague at **DevelopMentor** recently asked me about creating popup windows in XAML browser applications (XBAP). Normally this is not allowed - if you try to create a top-level window you will get a `SecurityException` because WPF asks for `UIPermission` which is strictly prohibited when hosted in the browser.

It turns out, however, that you _can_ get a popup window - there's a hidden little gem in the `System.Windows.Controls.Primitive` namespace that is your friend: `Popup`.

It's the same underlying class that ToolTip, Menu, and ComboBox use to display drop-down menus and overlays and it is browser-hosting aware! It's pretty limited in functionality - I'm not sure you can get it to move around with the mouse for example, but for simple cases it works great. Here's a code snippet - wire this up to a button in an XBAP:  

```csharp
void OnClick(object sender, EventArgs e)
{
    Popup window = new Popup();

    StackPanel sp = new StackPanel { Margin = new Thickness(5) };
    sp.Children.Add(new TextBlock { Text = "Hi from a popup" });

    Button newButton = new Button { Content = "Another button" };
    newButton.Click += delegate { window.IsOpen = false; };

    sp.Children.Add(newButton);
    sp.Children.Add(new Slider { Minimum = 0, Maximum = 50, Value = 25, Width = 100 });

    window.Child = new Border { Background = Brushes.White, BorderBrush = Brushes.Black, BorderThickness = new Thickness(2), Child = sp };
    window.PlacementTarget = this;
    window.Placement = PlacementMode.Center;
    window.IsOpen = true;
}
```

The key thing you need to do is set the `PlacementTarget` property. That associates a "parent" window and without it, the `Popup` class asserts `UIPermission` which will fail in the browser environment.
