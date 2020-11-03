---
title: 'MVVM: Service Locator'
date: 2010-01-27T00:00:00.0000000
layout: post
categories: mvvm
tags: mvvm dotnet
---

In this post, we’ll explore the service locator (called **ServiceProvider** in the library) and introduce the specific services included with the MVVM Helper library (as of 1.05 anyway). 

### What is the Service Locator?

In a nutshell, the service locator is a _resolver_ for services that your application might need or use.  It relates a type key to a specific object implementation.  This is nothing new – the service locator design pattern has been around forever and is considered a core J2EE pattern (see [ServiceLocator.html](http://java.sun.com/blueprints/corej2eepatterns/Patterns/ServiceLocator.html "http://java.sun.com/blueprints/corej2eepatterns/Patterns/ServiceLocator.html") for that definition).  In the MVVM Helpers library, this pattern is implemented in the **JulMar.Windows.Mvvm.ServiceProvider** class in **JulMar.Wpf.Helpers.dll**.  In retrospect, I’d prefer it be in a different namespace, but that’s where I stuck it initially.  Here’s what it looks like:

![ServiceProviderDefinition](/images/ServiceProviderDefinition_thumb.jpg "ServiceProviderDefinition")

Notice that it implements the **IServiceProvider** interface – this is a base .NET interface used for service resolution and a form of it can be found in the Visual Studio extensibility API, Windows Workflow and a variety of other projects.  The basis for the interface is you call the **GetService** method, passing a System.Type (generally an interface or abstract class type) and it returns the concrete implementation if the resolver knows about it.  This model allows the host to decide what the concrete implementation should be – for example in the Workflow world, the system needs to treat output completely differently for a Console App vs. a web application.  So, a console host might add a service for output that prints a string to the console window, while a web application might define the same service as a redirection to an output page.

### Ok, I get that, now how do I use it?

It’s easy.  From a client perspective, you simply request a service using the interface or class key.  As an example, in the previous post I mentioned the **IMessageVisualizer** interface which allows a ViewModel to popup a message box to notify the user about something.  Given that the VM is supposed to be somewhat technology agnostic and unit testable, we want to hide the implementation of displaying output behind an interface and that’s exactly what **IMessageVisualizer** is for.  To get the concrete implementation, we ask the **ServiceProvider**.  To do that, we need to define an instance of a provider – each instance has it’s own private collection of services that it manages.

For simplicity, the base **JulMar.Windows.Mvvm.ViewModel** class defines a public static **ServiceProvider** property:

```csharp
public class ViewModel : SimpleViewModel, IDisposable
{
    /// <summary>
    /// Service resolver for view models.  Allows derived
    /// types to add/remove
    /// services from mapping.
    /// </summary>
    public static readonly ServiceProvider ServiceProvider;
```
  
This property can be accessed anywhere in your project and it’s the specific instance that the default services are registered in (more on that in a second).  To get the message visualizer you could do this:

```csharp
var messageVisualizer = ViewModel.ServiceProvider.GetService(typeof(IMessageVisualizer)) as IMessageVisualizer;
if (messageVisualizer != null)
{
    // Use message visualizer
}
```
  
This invokes the **IServiceProvider.GetService** method to retrieve the service. Since that’s somewhat verbose, there is a typed helper method that cuts down your keystrokes and casts the return type automatically:

```csharp
var messageVisualizer = ViewModel.ServiceProvider.Resolve<IMessageVisualizer>();
```

As a shortcut, if you are deriving from the **ViewModel** base class it provides a convenience `Resolve<T>` method that cuts it to:

```csharp
var messageVisualizer = Resolve<IMessageVisualizer>();
```

Notice the null check to see if the service exists?  This is because services are dynamic, you register services with the service provider – so if the type key has not been registered the **Resolve** will return null to indicate an unsupported service.

### Service Registration

That brings us to registering and unregistering services.  This is accomplished through the **ServiceProvider.Add** and **ServiceProvider.Remove** methods:

```csharp
public class ServiceProvider : IServiceProvider
{
    /// <summary>
    /// Adds a new service to the resolver list
    /// </summary>
    /// <param name="type">
    /// Service Type (typically an interface)</param>
    /// <param name="value">Object that implements service</param>
    public void Add(Type type, object value);

    /// <summary>
    /// Remove a service
    /// </summary>
    /// <param name="type">Type to remove</param>
    public void Remove(Type type);
}
```

The **Add** method takes a type – you want to use an interface or abstract/virtual base class for extensibility, and an object that implements that interface/base class.

Once the type is registered, future calls to **GetService** (or **Resolve<T>**) will return the given object instance.  Note that some implementations of this pattern allow for lazy instantiation and per-call instance services (i.e. where you get a unique instance on each call to **GetService**).  I’ve not found a need for either of these and so this implementation is very simple.  Thinking about it, I’d probably register an instance that has a method to create the per-call items I need anyway.

The **Remove** method takes the type key and throws away the instance it is associated with – future calls made to retrieve the service will return null.

#### An Example: Spell Checking

Say for instance you needed a spell checking service for your application, and multiple ViewModels (and perhaps even view code behind files) will need access to it.  You could use the **ServiceProvider** to cache off your service and then access that instance as a singleton anywhere in your application – or even in other modules that are aware of the interface type.  That’s the idea behind this pattern, it allows you to loosely connect things together and bind them at runtime very easily.

So, first, I’d define a spell checking service interface (note this is an example so I’m keeping it _very_ simple):

```csharp
public interface ISpellCheckService
{
    bool CheckSpelling(string text);
}
```

Next, I’d have some concrete implementation of the above interface.  Let’s pretend I called it **DictionarySpellChecker**.  Then, somewhere in my application I add this service to the **ServiceProvider**, I might do this in my **Application** class:

```csharp
public partial class App
{
    public App()
    {
        // Register the spell checker service
        ViewModel.ServiceProvider.Add(typeof(ISpellCheckService), new DictionarySpellChecker());
    }
}
```
  
Then, any class that needs access to the spell check service simply needs to access the proper **ServiceProvider:**

```csharp
var spellChecker = ViewModel.ServiceProvider.Resolve<ISpellCheckService>();
```

#### Creating your own ServiceProvider instance

Note that here I’m using the ViewModel’s service provider – but that’s just because I know it’s there.  You can use the service provider independently of the ViewModel class.  All you need to do is:

1. Create an instance of the **ServiceProvider** class
1. Register your services against that specific instance
1. Make the instance available to anyone who needs the services (i.e. through a singleton property.

You can create as many service providers as you need to encapsulate your services.

### Built in services

The last thing I want to touch on are the built in services in the MVVM Helper library.  There are several services that can be registered, used or replaced in the library for your convenience.  In future blog posts I will detail each service specifically, but I’ll mention them here because they use the **ServiceProvider** as a resolver.

In the project template it registers all the services through a static method call – this is done in the Application constructor:

```csharp
public partial class App
{
    public App()
    {
        // Register the typical services (UI, Error, etc.)
        ViewModel.RegisterKnownServiceTypes();
           ...
   }
}
```
  
This call to **ViewModel.RegisterKnownServiceTypes** is required to use the built-in services; without it, none of them are registered.  If you look into the **ViewModel** source code, the definition for this method is:

```csharp
/// <summary>
/// This method registers known WPF services with the
/// service provider.
/// </summary>
public static void RegisterKnownServiceTypes()
{
   ServiceProvider.Add(typeof(IErrorVisualizer),
                       new ErrorVisualizer());
   ServiceProvider.Add(typeof(IMessageVisualizer),
                       new MessageVisualizer());
   ServiceProvider.Add(typeof(INotificationVisualizer),  
                      new NotificationVisualizer());
   ServiceProvider.Add(typeof(IUIVisualizer), new UIVisualizer());
   ServiceProvider.Add(typeof(IMessageMediator),
                       new MessageMediator());
}
```
  
As you can see, it’s just doing what what you would expect – registering a specific implementation for each service interface type.  The services are:

| Service | Description |
|---------|-------------|
| **IErrorVisualizer** | Presents an error dialog as a **MessageBox** with an OK button to dismiss it. |
| **IMessageVisualizer** | Presents a message dialog as a **MessageBox** with a selectable button set. |
| **INotificationVisualizer** | Allows long-running modal operations to present some wait notification (defaults to changing the cursor to an hourglass). |
| **IUIVisualizer** | Presents a custom modal or modeless dialog to the user associated with a **ViewModel**. |
| **IMessageMediator** | Message transfer service that allows loosely connecting **ViewModels** together. |

### Replacing services

Each service interface has a default implementation registered against it.  If you don’t like the implementation, or prefer to have your own custom implementation, simply add the service again _after it was registered_.

```csharp
public partial class App
{
    public App()
    {
        // Register the typical services (UI, Error, etc.)
        // for this application.
        ViewModel.RegisterKnownServiceTypes();
        // Replace the notification visualizer with our cool version..
        ViewModel.ServiceProvider.Add(typeof(INotificationVisualizer), new NotifyWaitBox());
   }
}
```
  
This will replace the default instance with a new provider – any future calls to retrieve the service will return your new instance.  This can be done at any time, and as many times as you like (as an example, consider a theme service you dynamically replace at runtime as the user selects the theme).

### Final Warnings

There is one warning I need to impart on the service locator.  It holds hard references to the registered instances.  That means if you want the instance to be Garbage Collected prior to the application terminating, you must remove the key from the service provider. This normally isn’t an issue as most services are lifetime services and are intended to be around until the end, but if you are using a transient service and only need it for a short period please remember this point so you don’t create an unintentional memory “leak”.

In the next post we’ll cover the Message, Error and Notification visualization services.  Stay tuned MVVM fans!
