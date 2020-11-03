---
title: 'Navigation and Persistence with MVVM in Windows Store Apps take #2'
date: 2013-04-26T00:00:00.0000000
layout: post
categories: mvvm
tags: mvvm uwp dotnet
---

Earlier I posted on an updated version of MVVMHelpers with persistence support. **Paulo Quicoli**, a long-time MVVMHelpers contributor, sent me a little code and a different way of managing navigation and persistence that he's been using. I thought it was quite elegant and asked if I could include a version in the library, which he graciously agreed to.  The result will be in the next release of the library - but I thought I'd introduce it here.

The idea is to use a serializer on the ViewModel so that it is always passed as the parameter in navigation - in this case, as a string.  The code Paulo provided was a simple use of the **DataContractJsonSerializer** to turn an object into a Json string - which we can then pass through the normal navigation APIs so it gets captured into the navigation stack automatically.  In this scenario, we are doing ViewModel-first - we'll create the ViewModel and then navigate to the view - passing the view model as the navigation parameter.

To accomplish this, two new classes were added into MVVMHelpers - both optional.  The first is the **JulMar.Windows.Serialization.Json** static class which looks like:

```csharp
public static class Json
{
    /// <summary>
    /// This method serializes an object or graph into a JSON string
    /// </summary>
    public static string Serialize(object instance);

    /// <summary>
    /// This takes a JSON string and turns it into an object or graph.
    /// </summary>
    public static T Deserialize(string stream);

    /// <summary>
    /// This takes a JSON string and turns it into an object or graph.
    /// </summary>
    public static object Deserialize(Type type, string stream);
}
```

This class is just a slim wrapper around the serializer to take an object and turn it into a string and vice-versa.  Note that the object must be serializable - which as of the last public release, the base **ViewModel** classes are.

The second class is an implementation of the **IPageNavigator** interface which supports the above serialization.  It does several things:

1. It uses the **NavigateTo** method which takes a page key (or type) and single parameter, or a parameter and ViewModel (in which case, the parameter is ignored).
1. It serializes the parameter/ViewModel into a string and passes that to the normal frame navigation API.
1. When the page is navigated TO, the string is de-serialized back into a ViewModel and set as the DataContext.
1. It properly handles suspension/resumption by saving/restoring the navigation stack and deserializing the ViewModel (if available) from the parameters back to the current page.

It's easy to use, although is not the default navigation provider - so there is a setup step that needs to be performed in order to use this new page navigator.  Specifically, in the startup sequence of the application (typically, the Application constructor in App.xaml.cs) you will need to add the highlighted line to replace the default page navigator:

```csharp
public App()
{
    ServiceLocator.Instance.Add(typeof(IPageNavigator), new AutoSerializingPageNavigator());
    this.InitializeComponent();
    this.Suspending += OnSuspending;
}
```

You still need to add the support to save/restore the state in your **OnLaunched** and **Resuming** events as well, this is the same code as the normal page navigator:

```csharp
protected async override void OnLaunched(LaunchActivatedEventArgs args)
{
    IPageNavigator pageNavigator = ServiceLocator.Instance.Resolve();

    Frame rootFrame = Window.Current.Content as Frame;
    if (rootFrame == null)
    {
        rootFrame = new Frame();
        Window.Current.Content = rootFrame;

        if (args.PreviousExecutionState == ApplicationExecutionState.Terminated)
        {
            await pageNavigator.LoadAsync();
        }
    }
    ...
}

private async void OnSuspending(object sender, SuspendingEventArgs e)
{
    IPageNavigator pageNavigator = ServiceLocator.Instance.Resolve();
    var deferral = e.SuspendingOperation.GetDeferral();
    await pageNavigator.SaveAsync();
    deferral.Complete();
}
```

Next, the ViewModel's must participate in this and be properly serializable - that means saving all the appropriate state by marking the properties with `[DataMember]` or with `[IgnoreDataMember]` based on whether you decorate the type with `[DataContract]` or not.  Also, remember that when you use `[DataContract]` the default constructor is not run - that means putting initialization logic into an `[OnDeseralizing]` method.  This is the primary reason I chose not to make this approach the default as it puts a bit of a burden onto the ViewModel - but once this is done, everything else is taken care of:

- It will navigate and setup the DataContext automatically
- It will properly save/restore the state on suspension/resumption

Here's an example of a simple color view model:

```csharp
[DataContract]
public sealed class ColorViewModel : ViewModel
{
    [DataMember] public string Color { get; set; }

    public IDelegateCommand GoBack { get; private set; }

    public ColorViewModel()
    {
        Initialize(new StreamingContext());
    }

    [OnDeserialized]
    void Initialize(StreamingContext context)
    {
        GoBack = new DelegateCommand(() => Resolve<IPageNavigator>().GoBack(),
            () => Resolve<IPageNavigator>().CanGoBack);
    }

    public ColorViewModel(string color) : this()
    {
        Color = color;
    }
}
```

And here's the logic to navigate to this view model:

```csharp
[DataContract]
public sealed class ColorViewModel : ViewModel
{
    [DataMember] public string Color { get; set; }

    public IDelegateCommand GoBack { get; private set; }

    public ColorViewModel()
    {
        Initialize(new StreamingContext());
    }

    [OnDeserialized]
    void Initialize(StreamingContext context)
    {
        GoBack = new DelegateCommand(() => Resolve<IPageNavigator>().GoBack(),
            () => Resolve<IPageNavigator>().CanGoBack);
    }

    public ColorViewModel(string color) : this()
    {
        Color = color;
    }
}
```

In addition, the Page and ViewModel can still participate in the **INavigationAware** interface - the new page navigator will still invoke it.  So this new capability allows for an alternative mechanism for managing page state and navigation - I expect to release it early next week! If you are interested in the full sample, check out [the Source Code](https://github.com/markjulmar/mvvmhelpers) and look at the sample test project [AutoSerializingNavigationTest](https://github.com/markjulmar/mvvmhelpers/tree/master/Metro/Test/AutoSerializingNavigationTest).
