---
title: Making Async Popups in WSAs
date: 2013-09-10T00:00:00.0000000
layout: post
categories: uwp
tags: uwp
---

The **async** and **await** keywords are awesome and allow for a much easier style of async programming over the traditional callback approach. Ideally, we could use these everywhere in our programming - particularly for UI environments such as Window Store apps. But not all the APIs are programmed this way in WinRT - take for example the **Popup** control.

You show the **Popup** by setting the **IsOpen** property to "true" and then close it by resetting it to "false". This is a holdover from the WPF/SL world where this was the paradigm used. The problem is that to manage a continuation of logic you now need to wire up to the **Closing** event which makes it harder to plug into MVVM style programming.

I had a question from a former student not long ago where he was trying to show a popup from a viewmodel but wanted to then process the results of the popup once it was dismissed - as part of the execution of an **ICommand** handler.  The two-fold nature of the API was getting in the way and he asked if there was a way to make it work more like a task-based API where async/await could be used.  It turns out that it's not hard at all - you can wrap the management into a simple method.

For example, if you have a **Button** in the UI to create a folder (like SkyDrive) and then in the handler (event or **ICommand**) do something like this:

```csharp
private async void OnShowPopup(object sender, RoutedEventArgs e)
{
    string filename = await GetFolderName();
    if (filename != null)
    {
        await new MessageDialog(filename, "Got:").ShowAsync();
    }
}
```

Notice the call to **GetFolderName** - it shows the **Popup** and then provides the returned folder name. This code could easily be in a viewmodel (the **MessageDialog** not withstanding of course). Here's the **GetFolderName** method:

```csharp
private Task<string> GetFolderName()
{
    string theFilename = null;
    ManualResetEventSlim evt = new ManualResetEventSlim();
    Popup popup = new Popup {
        IsLightDismissEnabled = true,
        HorizontalOffset = Window.Current.Bounds.Width/2,
        VerticalOffset = Window.Current.Bounds.Height/2
    };

    TextBox tb = new TextBox() {
        Margin = new Thickness(5),
        Padding = new Thickness(5)
    };

    Button btn = new Button() {
        Content = "Create",
        Margin = new Thickness(5),
        IsEnabled = false
    };

    btn.Click += (s, e) => { theFilename = tb.Text; popup.IsOpen = false; };
    tb.TextChanged += (s, e) => { btn.IsEnabled = !string.IsNullOrEmpty(tb.Text); };

    Border bd = new Border() {
        Background = new SolidColorBrush(Colors.Black),
        BorderBrush = new SolidColorBrush(Colors.White),
        BorderThickness = new Thickness(2),
        Width = 300,
        Padding = new Thickness(20)
    };

    StackPanel sp = new StackPanel();
    sp.Children.Add(new TextBlock() { Text = "Enter folder name" });
    sp.Children.Add(tb);
    sp.Children.Add(btn);
    bd.Child = sp;

    popup.Child = bd;
    popup.Closed += (s, e) => evt.Set();
    popup.IsOpen = true;
    tb.Focus(FocusState.Programmatic);

    return Task.Run(() => { evt.Wait(); return theFilename; });
}
```

Here we are building the **Popup** dynamically - you could just as easily provide XAML for this (particularly for the **Content**). In this case, we wire into the **Closed** event and signal a thread to return the collected filename. Then a **Task** is spun up to wait for the signal and this is what the UI is going to use await on. In this way, we can use the same sort of logic we might use with an I/O based API in our viewmodel but with a **Flyout** or **Popup**.
