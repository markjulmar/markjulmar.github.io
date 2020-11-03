---
title: MVVM Helpers 2.0 is live
date: 2010-04-16T00:00:00.0000000
layout: post
categories: mvvm
tags: mvvm dotnet
---

... and [available on GitHub](https://github.com/markjulmar/mvvmhelpers).

There are several new features in this release that I've been tinkering with for a while.  First, I now use MEF to link things together.  I waited until .NET 4.0 was released to push this out because MEF is part of the framework now.  If you don't want to take a dependency on it (in 3.5) then please stay with 1.06 which is also available. 

### Managing Services with the Service Locator pattern + MEF

The advantage to MEF are several.  First, you can easily add or replace services.  To add a new service to the service locator, you can just add your class and then decorate it with `[ExportServiceProvider]`:

```csharp
[ExportServiceProvider(typeof(PlaySoundService))]
public class PlaySoundService
{
    public void Beep()
    {
        new SoundPlayer(@"./media/ding.wav").Play();
    }
}
```

There’s no need to register it, and if you want to replace an existing service (such as the `IUIVisualizer`) you can simply supply the interface type to the attribute – the `ServiceProvider` will automatically defer to your version instead of the built-in one.  This also means there’s no need to register default services – so the `RegisterDefaultServices` call is gone.

To consume the service you can use the traditional syntax:

```csharp
public MainViewModel()
{
    BeepCommand = new DelegatingCommand(() => Resolve<PlaySoundService>().Beep());
}
```

or you can rely on MEF to do dependency injection if you’d like to decouple from the service provider altogether.  To do this, simply declare a field or property and mark it with an `[Import]` attribute, and make sure to add an `[Export]` to the service declaration:

```csharp
[ExportServiceProvider(typeof(PlaySoundService))]
[Export]
public class PlaySoundService
{
}
...

public class MainViewModel : ViewModel
{
   [Import] private PlaySoundService _playSoundService;

   public ICommand BeepCommand { get; private set; }

   public MainViewModel()
   {
      BeepCommand = new DelegatingCommand(() => **_playSoundService.Beep()**);
   }
}
```

How cool is that?  All the pre-defined services are already exported through the interface itself, so if you want to import a built-in service such as the `IUIVisualizer`, you can simply create a field and mark it as an `[Import]` and MEF will take care of the rest.  As I mentioned, you can continue to use `Resolve<T>` if you prefer a more obvious approach, or like me, prefer to not cache off common services in fields.  Be aware that the instance supplied to you for the predefined services will always be global – i.e. you will get a copy of the **exact same instance** that the service provider hands out through `Resolve<T>`.

### View First Creation with MEF

The next new thing is VM + View creation.  The library supports both ViewModel first and View first creation.  The MainWindow is typically created using View-first mechanics (since it is normally created by the application startup process).  To hook up a ViewModel to it you use the `ViewModelCreator` markup extension, in prior versions of the library you supplied the type to create (and optionally the designer type):

```xml
<Window x:Class="WpfMvvmApplication1.Views.MainWindow"
   xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
   xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
   xmlns:julmar="http://127.0.0.1:8080/wpfhelpers"
   xmlns:ViewModels="clr-namespace:WpfMvvmApplication1.ViewModels"
   DataContext="{julmar:ViewModelCreator {x:Type ViewModels:MainViewModel}, DesignerViewModelType={x:Type ViewModels:.DesignTimeViewModel}}"
   Title="Test Project" Height="400" Width="500">
```

Notice that this requires we push knowledge of the VM into the view – this isn’t necessarily a bad thing and you can still use this approach, but Version 2.0 adds in the ability to decouple them further.  Now you can specify the shared `Key` to use to connect them together.  If you add this, the type (if supplied) will be ignored:

```xml
<Window x:Class="WpfMvvmApplication1.Views.MainWindow"
   xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
   xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
   xmlns:julmar="http://127.0.0.1:8080/wpfhelpers"
   DataContext="{julmar:ViewModelCreator Key=MainWindow}"
   Title="Test Project" Height="400" Width="500">
```

The key is just a shared string – so it’s not compile-time safe which is a downside to this approach.  However now we have no dependency at all on the type being created by the developer.  A design-time DLL could supply any type as long as it exports the proper key.  Here’s how we associate the ViewModel:

```csharp
[ExportViewModel("MainWindow")]
public class MainViewModel : ViewModel
{
```

Note the shared key being used here.  With this in place, the **ViewModelCreator** extension will go through MEF to locate and create the ViewModel.  Any imports/exports on that type would be satisfied as normal.

### ViewModel First creation with MEF + IUIVisualizer

You can also go the other direction – where the VM is created first and you want to associate the view dynamically.  The approach I’ve taken here is to piggy back on the existing `IUIVisualizer` service.  Recall from prior posts that it is responsible for creating new popup windows (either modal or modaless) inside the VM – to display child dialogs for example.  Prior to this release, you were responsible for registering each view with the service.  With the magic of MEF you can now simply decorate the View with `[ExportUIVisualizer]`:

```csharp
[ExportUIVisualizer("ChildWindow")]
public partial class ChildWindow : Window
{
```

This will cause the window to be registered automatically with the `IUIVisualizer` service so that in some VM you can show it:

```csharp
ShowChild = new DelegatingCommand(() => Resolve<IUIVisualizer>().ShowDialog("ChildWindow"));
```

Again, the key is a string that you must synchronize, not unlike the registration process in V1 of the library.  You can pass the ViewModel into the child dialog (**Show** and **ShowDialog** both take a parameter for this), or if you place the {ViewModelCreator} tag onto your child window it will be located dynamically.

### Closing Thoughts

Finally, another minor change is the base ViewModel class now automatically registers with the message mediator, so the call to register is not necessary in derived classes.  It has been removed so any existing code will get a compile error if you try to call it – just remove your call.

I hope you find the library easy to use – the source, binaries and templates are all on GitHub.  There is a project template for Visual Studio 2008 and another for Visual Studio 2010 – make sure to copy them into the appropriate template directories.  As always, comments are welcome!

Here’s the sample if you want to peruse it:  [V2SampleApp.zip](/samples/V2SampleApp.zip)
