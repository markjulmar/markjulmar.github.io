---
title: Using the Message Mediator Service in MVVM Helpers
date: 2011-02-22T00:00:00.0000000
layout: post
categories: mvvm
tags: mvvm dotnet
---

One of the services exposed in the MVVM Helpers library is a Message Mediator.  This service implements the [mediator design pattern](http://www.dofactory.com/Patterns/PatternMediator.aspx) which is used to enable objects to talk to each other without having any explicit knowledge of each other.  In this pattern, we use a third-party object (the “mediator”) to negotiate communication between the interested parties.

The message mediator is actually part of the **JulMar.Core** assembly – this is a general .NET assembly that contains useful code, extensions and services which are not specific to the presentation technology being used (i.e. they can be used in any .NET project type).  I actually work in all kinds of fields and technologies, but WPF has a special place in my heart – and it turns out this service is pretty useful for GUI applications to enable loose-coupling between elements, particularly when you are relying on other design patterns such as Model-View-ViewModel (MVVM).

To introduce the service, let’s look at the definition of the interface that describes it in **JulMar.Core.Interfaces**:

```csharp
///<summary>
/// The interface definition for our Message mediator.  This allows loose-event coupling between components
/// in an application by sending messages to registered elements.
///</summary>
public interface IMessageMediator
{
    /// <summary>
    /// This registers a Type with the mediator.  Any methods decorated with <seealso cref="MessageMediatorTargetAttribute"/> will be 
    /// registered as target method handlers for the given message key.
    /// </summary>
    /// <param name="view">Object to register</param>
    void Register(object view);

    /// <summary>
    /// This method unregisters a type from the message mediator.
    /// </summary>
    /// <param name="view">Object to unregister</param>
    void Unregister(object view);

    /// <summary>
    /// This registers a specific method as a message handler for a specific type.
    /// </summary>
    /// <param name="key">Message key</param>
    /// <param name="handler">Handler method</param>
    void RegisterHandler<T>(string key, Action<T> handler);

    /// <summary>
    /// This registers a specific method as a message handler for a specific type.
    /// </summary>
    /// <param name="handler">Handler method</param>
    void RegisterHandler<T>(Action<T> handler);

    /// <summary>
    /// This unregisters a method as a handler.
    /// </summary>
    /// <param name="key">Message key</param>
    /// <param name="handler">Handler</param>
    void UnregisterHandler<T>(string key, Action<T> handler);

    /// <summary>
    /// This unregisters a method as a handler for a specific type
    /// </summary>
    /// <param name="handler">Handler</param>
    void UnregisterHandler<T>(Action<T> handler);

    /// <summary>
    /// This method broadcasts a message to all message targets for a given
    /// message key and passes a parameter.
    /// </summary>
    /// <param name="key">Message key</param>
    /// <param name="message">Message parameter</param>
    /// <returns>True/False if any handlers were invoked.</returns>
    bool SendMessage<T>(string key, T message);

    /// <summary>
    /// This method broadcasts a message to all message targets for a given parameter type.
    /// If a derived type is passed, any handlers for interfaces or base types will also be
    /// invoked.
    /// </summary>
    /// <param name="message">Message parameter</param>
    /// <returns>True/False if any handlers were invoked.</returns>
    bool SendMessage<T>(T message);

    /// <summary>
    /// This method broadcasts a message to all message targets for a given
    /// message key and passes a parameter.  The message targets are all called
    /// asynchronously and any resulting exceptions are ignored.
    /// </summary>
    /// <param name="key">Message key</param>
    /// <param name="message">Message parameter</param>
    void SendMessageAsync<T>(string key, T message);

    /// <summary>
    /// This method broadcasts a message to all message targets for a given parameter type.
    /// If a derived type is passed, any handlers for interfaces or base types will also be
    /// invoked.  The message targets are all called asynchronously and any resulting exceptions
    /// are ignored.
    /// </summary>
    /// <param name="message">Message parameter</param>
    void SendMessageAsync<T>(T message);
}
```

As with most things in MVVM Helpers, the above interface has a private, internal implementation that is used by default – you an override services by supplying your own implementation of the above interface in your code.  It will get picked up automatically by the library (I use MEF to pair up dependencies) assuming you go through the service locator (**IServiceProvider**).  I’ll show an example of getting the service in a moment.

If you examine the interface, most of the methods relate to actually sending messages and then there are a couple of methods used for registration and de-registration with the mediator.  The point of this service is to loosely couple things together, to accomplish that, the two endpoints both take a dependency against the mediator service – one side registers with the mediator to receive traffic, the other side sends traffic.  If both sides register you can perform bi-directional messaging.  All the message sent by the built-in implementation simply use delegates to wire things together – so it’s all in-AppDomain messaging.  If you need to go cross-AppDomain, cross-process, or cross-machine, consider using a formal WCF service or a message queue implementation.

To start off with, let’s look at the registration APIs.  There are three basic forms:

```csharp
void Register(object view);  
void RegisterHandler<T>(string key, Action<T> handler);  
void RegisterHandler<T>(Action<T> handler);
```

Basically, there are two types of registration – direct (**RegisterHandler**), and attributed (**Register**).  Both forms accomplish exactly the same thing, just in different ways. 

**RegisterHandler** is useful if you have a single method you want to have as a target for a message, or if you have an anonymous method (or lambda) you want to assign as the target of a message.  It has two forms – both take an `Action<T>` handler which is the actual target body.  One of the forms also takes a string **key**.

The message mediator service provided here has two ways of sending messages – one involves just passing a typed data structure, the other involves passing a data structure and a key to identify the target.  The key is always a string (for simplicity), and can be anything you like but the sender and receiver have to agree on the string and the case must match.  When no key is supplied, the object data type is used as the key – so anytime that specific type of object is used as the parameter for a message, registered targets expecting that type of object will be notified.

The **Register** method is a bit more mysterious in terms of how it works.  It allows multiple subscriber endpoints to be registered on an object as a group.  The single parameter is the object implementing the subscriber (target) methods.  Each method must be decorated with a `[MessageMediatorTarget]` attribute (at least for the built-in implementation, if you replaced it then you could use whatever convention you like).

There are also paired un-register methods:

```csharp
void Unregister(object view);  
void UnregisterHandler<T>(string key, Action<T> handler);  
void UnregisterHandler<T>(Action<T> handler);  
```

If the lifetime of the publisher / subscriber are different then you should make sure to unregister from the mediator to allow the subscriber to be efficiently collected.  The built-in implementation of the service doesn’t explicitly require this at it holds onto weak references to the delegates and instances so they will be released when they are no longer necessary.  This also means any lambda you use must be an instance lambda or it might go away unintentionally – make sure you hold references to your subscribers while they are to be alive (make them static if you want global lifetime for example).

The other methods on the interface are for the publisher and revolve around _sending_ messages.

```csharp
bool SendMessage<T>(string key, T message);  
bool SendMessage<T>(T message);  
void SendMessageAsync<T>(string key, T message);  
void SendMessageAsync<T>(T message);  
```

Notice there are overrides for using a typed message with and without a key, and there are async versions available which will send the message on a background thread.  The default behavior with **Send** is to send on the caller thread and block the caller while the message is being processed.  These synchronous versions return a true/false result indicating if a target processed the message.  The async forms do not have any (easy) way to return a result.

Let’s get to some examples to make this a little more concrete.  First, I am going to create a new WPF 4.0 project in Visual Studio 2010 – I’ll name it **MessageMediatorExamples** for clarity (the code is at the end of the article if you’d like to download the sample).  Once it’s done creating, I’ll add the required libraries through NuGet ([www.nuget.org](http://www.nuget.org)) by right-clicking on the solution and selecting “Add Library Package Reference”:

![image](/images/using-the-message-mediator-service-in-mvvm-helpers-image_thumb.png "image")

If you search in the packages for MVVMHelpers you will find it:

![image](/images/using-the-message-mediator-service-in-mvvm-helpers-image_thumb_1.png "image")

Install the package and it adds the required assemblies for you.  Pretty cool stuff – easy to use. 

To show off the mediator, I setup the **MainWindow** to have a **TextBox** as it’s primary content – I then created a **ViewModel** to hold the data for it:

```csharp
public class MainViewModel : ViewModel  
{  
    private string _text, _font, _foregroundColor, _backgroundColor;  
    private double _fontSize;  
  
    public string Text  
    {  
        get { return _text; }  
        set { _text = value; OnPropertyChanged(() => Text); }  
    }  
  
    public string Font  
    {  
        get { return _font; }  
        set { _font = value; OnPropertyChanged(() => Font); }  
    }  
  
    public string ForegroundColor  
    {  
        get { return _foregroundColor; }  
        set { _foregroundColor = value; OnPropertyChanged(() => ForegroundColor); }  
    }  
  
    public string BackgroundColor  
    {  
        get { return _backgroundColor; }  
        set { _backgroundColor = value; OnPropertyChanged(() => BackgroundColor); }  
    }  
  
    public double FontSize  
    {  
        get { return _fontSize; }  
        set { _fontSize = value; OnPropertyChanged("FontSize"); }  
    }  
  
    public MainViewModel()  
    {  
       ...  
    }  
}
```

I set this view model as the DataContext for the window in XAML and data bound the properties of the `TextBox` to the View Model properties:

```xml
<Window x:Class="MessageMediatorExamples.MainWindow"  
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"  
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        xmlns:ViewModels="clr-namespace:MessageMediatorExamples.ViewModels"
        Title="MainWindow" mc:Ignorable="d" xmlns:d="http://schemas.microsoft.com/expression/blend/2008"   
        xmlns:mc="http://schemas.openxmlformats.org/markup-compatibility/2006" d:DesignHeight="300" d:DesignWidth="400">  

    <Window.DataContext>  
        <ViewModels:MainViewModel />  
    </Window.DataContext>  

    <DockPanel Background="DarkOliveGreen">  
        <TextBox Margin="5" AcceptsReturn="True" TextWrapping="Wrap"  
                 VerticalScrollBarVisibility="Auto"  
                 Text="{Binding Text}"   
                 Foreground="{Binding ForegroundColor}"  
                 Background="{Binding BackgroundColor}"  
                 FontFamily="{Binding Font}"   
                 FontSize="{Binding FontSize}"/>  
    </DockPanel>  
</Window>
```
  
Running the application produces the expected results:

![image](/images/image_thumb_4.png "image")

Next, I added two more windows to the project – one to hold fonts and the other to hold colors.  Since this post is about the mediator, I’ll leave the implementation of these to your imagination (or to looking at the final sample!)

Our goal is to link the three of these together using the message mediator.  As a first step, let’s add some support to invoke the dialogs.  Users of MVVMHelpers are aware that it includes a visualization service for this, but I’m not going to use it here – instead, let’s send a message to the view from the ViewModel to actually create a secondary view bound to the same view model.  We’ll do this for both dialogs – to actually invoke the dialog we’ll use commands bound to two toolbar buttons:

```xml
<ToolBar DockPanel.Dock="Top">  
    <Button Content="Colors" Command="{Binding ChangeColors}" />  
    <Button Content="Fonts" Command="{Binding ChangeFonts}" />  
</ToolBar>  
```
  
.. and in the View Model, create the command handlers using the **DelegatingCommand** object:

```csharp
public ICommand ChangeColors { get; private set; }  
public ICommand ChangeFonts { get; private set; }  
  
public MainViewModel()  
{  
    ...  
    ChangeColors = new DelegatingCommand(OnChangeColors);  
    ChangeFonts = new DelegatingCommand(OnChangeFonts);  
}  
  
private void OnChangeColors()  
{  
}  
  
private void OnChangeFonts()  
{  
}  
```

Now, when we handle the commands, we’ll send a message to the main view to create the appropriate modaless dialogs for us – passing our view model as a parameter, and using a string key to identify the action.  When using string-based keys, I prefer to consolidate the strings to a known class – here I’ll call it **ViewMessages**:

```csharp
public static class ViewMessages  
{  
    public const string ChangeColors = "ChangeColors";  
    public const string ChangeFonts = "ChangeFonts";  
}  
```

Then, I’ll call the built-in **SendMessage** method I inherit from the **ViewModel** base class to send the message to the mediator, note that if you derive from SimpleViewModel, or are not in the ViewModel inheritance change then you would need to get the **IMessageMediator** implementation either through dependency injection or through the service locator service – more on that in a second.

```csharp
private void OnChangeColors()  
{  
    this.SendMessage(ViewMessages.ChangeColors, this);  
}  
  
private void OnChangeFonts()  
{  
    this.SendMessage(ViewMessages.ChangeFonts, this);  
}  
```

Now, we need a handler for both of these messages.  Since I want to perform a view-specific action here that’s not testable, the clear place to put this logic is into the View, so let’s switch to the MainWindow.xaml.cs and add a handler for the message.  Recall there are two ways to register a handler – through attributes, or through direct registration.  We’ll use both to show how they work.  In order to register handlers, we need access to the message mediator.  The easiest way to get the instance is to ask for it:

```csharp
public partial class MainWindow  
{  
    private IMessageMediator _messageMediator;  
  
    public MainWindow()  
    {  
        _messageMediator = ViewModel.ServiceProvider.Resolve<IMessageMediator>();  
  
        InitializeComponent();  
    }  
}  
```

You could also add an `[Import]` attribute and then ask MEF to compose the view – that would inject the implementation into the view, but for simplicity, let’s pull the value.  Next, let’s register a handler for the Color message:

```csharp
public MainWindow()  
{  
    _messageMediator = ViewModel.ServiceProvider.Resolve<IMessageMediator>();  
  
    _messageMediator.RegisterHandler<MainViewModel>(ViewMessages.ChangeColors, OnChangeColors);  
  
    InitializeComponent();  
}  
  
private void OnChangeColors(MainViewModel vm)  
{  
      
}  
```

Notice how we get type safety by using the generic version?  If you want to handle _any_ object as the parameter you can use the non-generic form of the **RegisterHandler** as well.  This will call the target method with a general object type vs. requiring it to be a MainViewModel as shown here.

The implementation will show the color dialog – setting it’s `DataContext` to be the passed view model.  For the second (font) window, we’ll use the attribute syntax – this involves adding a **[MessageMediatorTarget]** attribute to the target method:

```csharp
[MessageMediatorTarget(ViewMessages.ChangeFonts)]  
private void OnChangeFonts(MainViewModel vm)  
{  
    FontWindow window = new FontWindow() {DataContext = vm};  
    window.Show();  
}  
```

Now, in order to make this work, I need to also call the Register method – passing my view instance – we’ll do this in the constructor:

```csharp
_messageMediator.Register(this);  
```

This will use reflection on the instance and identify all instance methods with the attribute applied – it will then call **RegisterHandler** for each one.  With the magic of Data Binding (remember the child dialogs are sharing the main view model – and they have their selections bound to the same properties being used by the TextBox), we get simple font and color changes applied easily:

![image](/images/image_thumb_5.png "image")

You can also send messages using a typed data structure.  For example, if we wanted to encapsulate all the font / color information into a single dialog and then use a child view model to drive it we could do this:

```csharp
class FontColorInfo  
{  
   public string ForegroundColor {get;set;}  
   public string BackgroundColor {get;set;}  
   public string Font {get;set;}  
}  
...  
  
void OnChangeColor()  
{  
   this.SendMessage(new FontColorInfo() { ... });  
}  
  
...  
  
[MessageMediatorTarget]  
void OnChangeFontAndColors(FontColorInfo fci)  
{  
   .. invoke dialog using passed data ..  
}
```
  
Notice that now we don’t need magic strings – but it requires you have a devoted data structure which is then used as the key for the message target lookup.  Another thing you can use the mediator for is to do dynamic queries – targets are called in sequence and can modify the passed data.  When the `SendMessage` call returns, the data could be filled in:

```csharp
void OnGetLoadedFiles()  
{  
   List<string> filenames;  
   SendMessage("GetLoadedFilesFromAllViews", filenames);  
   foreach (string file in filesnames)  
   {  
     ...  
   }  
}
```

Here we pass a message to all registered targets – each target (called sequentially) would add it’s loaded files to the passed `List<string>`.  When the message returns the code can process all the files in one operation.  It’s a nice, loosely coupled way of collecting data from various sources.

The final project for this blog post is located [here](/samples/MessageMediatorExamples.zip)
