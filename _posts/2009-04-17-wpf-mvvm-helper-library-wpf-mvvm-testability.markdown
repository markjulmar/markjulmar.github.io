---
title: WPF MVVM Helper Library (WPF + MVVM = testability)
date: 2009-04-17T00:00:00.0000000
layout: post
categories: mvvm
tags: mvvm wpf dotnet
---

There's been a lot of talk about the Model-View-ViewModel pattern recently and it's usage around the WPF and Silverlight technology stack.  When teaching WPF, I always introduce students to MVVM as part of the Essential WPF class, it's an incredibly useful pattern that really separates the UI from the code behind behavior.  One of the things I give the students is a library to do MVVM - I also use it in my consulting work.  With all the focus on it lately, I figured maybe it's time to release it to the public.

A bit of history -- the library is really just a place where I dump all kinds of useful utility classes, helpers, wrappers, etc. that I tend to use a lot.  I started it about 3 years ago and it wasn't originally intended to be just an MVVM implementation so you'll find it's got all kinds of stuff in it, not all of which is MVVM specific.  It's evolution owes a lot to various blog posts, WPF Disciples, and other WPF leaders; I certainly didn't invent anything radically new but borrowed heavily from all kinds of places as I built various classes I needed for my own work.  These classes tended to evolve with new functionality (either due to necessity, or because a good idea occurred to me or someone else). For example, there was a recent thread on the Mediator pattern (initiated by Marlon Grech and added on by Josh Smith, Laurent Bugnion and others); I already had a message mediator in place but the idea of using an attribute to hook it up was a great one that I adopted into my library just because I like the idea.  The Delegating command pattern is one you see in a lot of places - including the Prism implementation.  The event routing attached behavior was made possible by a couple of blog posts by Mike Hilberg and John Gossman. So, be aware that as you use this code, it owes a lot to a variety of people.  That said, any bugs or issues are completely mine and I take full credit for them.

So, what all is here?  Well, quite a bit.  As I said, this is a collection of helpers I've built and reused over the years doing WPF consulting and instruction.  When MVVM came to my notice I worked on trying to completely separate the XAML side so I'll focus on those classes here. 

The basic idea is to derive your ViewModel classes from one of three base classes depending on what you need:

- `JulMar.Windows.Mvvm.ViewModel` - supports basic `INotifyPropertyChanged` and Close/Activate eventing to the view.  
- `JulMar.Windows.Mvvm.ValidatingViewModel` - supports everything above, adds Validation through `IDataErrorInfo` support  
- `JulMar.Windows.Mvvm.EditingViewModel` - supports basic + validation + `IEditableObject`
  
Next, in each view (XAML) you set the `DataContext` property to your view model, the library has a `MarkupExtension` to do the work for you -

```xml
<Window x:Class="TestMvvm.MainWindow" xmlns:me="clr-namespace:TestMvvm"  
         DataContext="{julmar:ViewModelCreator ViewModelType={x:Type me:WinViewModel}}">
```

This creates the view and also wires up support for closing the view and activating the view through the ViewModel class (using the **RaiseCloseRequest** and **RaiseActivateRequest** methods).

Everything is driven off `ICommand` - you can bind commands to the lifetime of the view (so you can detect activation, deactivation, loading, closing) through the **LifetimeEvents** attached behavior:

```xml
<Window x:Class="TestMvvm.MainWindow"  
        julmar:LifetimeEvent.Activated="{Binding ActivatedCommand}"
        julmar:LifetimeEvent.Close="{Binding CloseCommand}"
        julmar:LifetimeEvent.Loaded="{Binding LoadedCommand}"
        julmar:LifetimeEvent.Deactivated="{Binding DeactivatedCommand}" >
```

These execute the bound command in the ViewModel when those events occur in the view.  If you need other events styles you can use the EventCommander attached behavior which allows any arbitrary event to be wired to a command.  This can be placed on any `UIElement`:

```xml
<julmar:EventCommander.Mappings>
    <julmar:CommandEvent Command="{Binding MouseEnterCommand}" Event="MouseEnter" />
    <julmar:CommandEvent Command="{Binding MouseLeaveCommand}" Event="MouseLeave" />  
<julmar:EventCommander.Mappings>
```

You can also wire up keyboard and mouse gestures to commands using the **InputBinder** attached property:

```xml
<julmar:InputBinder.Bindings>  
    <julmar:KeyBinding Command="{Binding ExitCommand}" Key="F3" Modifiers="ALT" />
    <julmar:MouseBinding Command="{Binding ExitCommand}" Gesture="Control+RightClick" />  
<julmar:InputBinder.Bindings>
```

This replaces the traditional **InputBindings** collection with a data bindable version - it also supports **CommandParameter** objects on each **KeyBinding** or **MouseBinding**.  You can use it on any element which supports input bindings.

There are some helper classes implemented through interfaces and a (very) simple **ServiceProvider** to locate them:

- `IErrorVisualizer` - to display errors (title + message).  Default implementation displays `MessageBox` with OK button.  
- `IMessageVisualer` - to display messages with a prompt.  Default implementation uses `MessageBox`.  
- `INotificationVisualizer` - to display a wait prompt.  Default implementation uses an hourglass cursor.  
- `IUIVisualizer` - to display other views through a key.  Supports both modal and modaless display.

You use the **ServiceProvider** to find the services - the base `ViewModel` class has support built in for it:

```csharp
Resolve<IErrorVisualizer>().Show("An Error Occurred", e.Description);
```

You can register the default implementation for all of the above in the **App.xaml.cs** file -

```csharp
ViewModel.RegisterKnownServiceTypes();
```

You can also provide your own implementation through the **ServiceProvider** itself.  The **ViewModel** class has a public static field:

```csharp
ServiceProvider.Add(typeof(IErrorVisualizer), new MyErrorVisualizer());
```

This replaces or adds the given service (using the type as the key) to the registry database.  You then use **Resolve** to find the service at runtime in any view.  Creating secondary views is done through the **IUIVisualizer** interface.  The default implementation provides a registry and must be added explicitly to use it:

```csharp
IUIVisualizer controller = new UIVisualizer(
            new Dictionary {  
                {Dialogs.AddNewPage, typeof(AddNewPageWindow)},  
                {Dialogs.ManagePages, typeof(BrowseWindow)},  
                {Dialogs.NewLogon, typeof(LoginDialog)}  
            });

ServiceProvider.Add(typeof(IUIVisualizer), controller);
```

This adds three UI dialogs to the visualizer - the key is a simple string, the second parameter is a **Type** object that represents the **Window** to create.  You then get the window to display through the **IUIVisualizer** interface:

```csharp
LoginViewData ld = new LoginViewData();

IUIVisualizer uiController = ServiceProvider.Resolve();  
if (uiController.ShowDialog(Dialogs.NewLogon, ld).Value == true)  
{  
}
```

This shows the login dialog, using the **LoginViewData** as the **DataContext** (ViewModel).  It displays it as a modal dialog and returns the final result.  You can also display modaless dialogs which take an optional completion proc:

```csharp
public bool Show(string key, object state, bool setOwner, EventHandler completedProc)
```

The completed procedure is invoked when/if the window ever closes.  Again, the state passed is the data context and if it derives from ViewModel the appropriate Closing/Activated events will be wired up to the events in the ViewModel.

I've been using this library for quite a while and it works great (for me).  I provide it here with full source code so you can diagnose any issues you have, or just look through it.  Feel free to use it however you like.  I don't claim this to be the end-all implementation - as I said much of the ideas expressed in here can be found elsewhere.  There is also a help file provided and a simple example of how to use some of the classes provided that I whipped up for fun.

[Here's the GitHub repo with all the code and samples](https://github.com/markjulmar/mvvmhelpers)

Enjoy - let me know if you have any issues or think of a good addition!  Just remember I put no guarantee on this code - consider it a sample for you to do whatever you like with.
