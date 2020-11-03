---
title: MVVMHelpers.Metro updated
date: 2013-03-13T00:00:00.0000000
layout: post
categories: mvvm
tags: mvvm dotnet
---

The Metro version of MVVMHelpers was updated yesterday with some key new features, one of them is some built-in state management for process lifetime and navigation. This is done through two interfaces - **IPageNavigator** and **IStateManager**.

You can use them independently, or together. By default, if you don't assign one, the PageNavigator service will internally create a StateManager to store off page navigation state. You can supply your own IStateManager implementation by either overriding the framework (i.e. apply an **[Export(typeof(IStateManager)]** to a type) or by simply setting the StateManager property on the IPageNavigator itself. I'll show that in a later blog post, but now let's see how they work together. To give you an example, let's build a sample app.

First, fire up Visual Studio 2012 on a Windows 8 box and create a new Blank App project. I called it "SampleBoxes". Next, add references to MVVMHelpers and MEF through Nuget: Right click on references and select "Manage Nuget References". Select "Online" and type "MVVMHelpers" in the search box:

![](/images/nuget_pic1-300x200.jpg "nuget_pic1")

Click "Install" to add the references. Next, modify the app.xaml.cs with code to use the new types - you can use the ServiceLocator to get the IPageNavigator interface.

```csharp
protected override void OnLaunched(LaunchActivatedEventArgs args)
{
    IPageNavigator pageNavigator = ServiceLocator.Instance.Resolve();
    Frame rootFrame = Window.Current.Content as Frame;
    ...
}
```

In the same method, just a little further down, modify the code to create the frame, assign the content and then reload any persisted settings. This must be done _after_ setting the frame as this method both loads the underlying state using an internal IStateManager and also restores the navigation stack if it is present. **Note: since the call to LoadAsync is asynchronous, we will use an await here, make sure to mark the method itself as an async method**

```csharp
if (rootFrame == null)
{
    // Create a Frame to act as the navigation context and navigate to the first page 
    Window.Current.Content = rootFrame = new Frame();

    // Reload our saved state
    if (args.PreviousExecutionState == ApplicationExecutionState.Terminated)
    {
        await pageNavigator.LoadAsync();
    }
}
```

Finally, in the `OnSuspending` event handler at the bottom of the **app.xaml.cs** file, locate the `IPageNavigator` and call `SaveAsync` to save off all the page state and navigation stack.

```csharp
private async void OnSuspending(object sender, SuspendingEventArgs e)
{
    var deferral = e.SuspendingOperation.GetDeferral();
    IPageNavigator pageNavigator = ServiceLocator.Instance.Resolve();
    await pageNavigator.SaveAsync();
    deferral.Complete();
}
```

That's all we need to add in order to save and restore our state. Now, how do we actually add things to the state manager itself? The key is the INavigationAware interface - if you implement this in either the View (`Page` classes) or ViewModel (`DataContext` of the Page) then it will automatically get called when the page navigation occurs, both from and to. The event args will have an `IDictionary` which you can use to store off any settings specific to that page or view model. Note that the dictionaries will be different for each (they are segregated so you do not have to worry about key overlaps).

Create a folder in the solution called "ViewModels" and add a new class into the folder called MainViewModel.cs, go ahead and derive from the ViewModel base class. Next, add a data class into the project - I've named it `ColorBox` and here's the code:

```csharp
public sealed class ColorBox
{
    public string Color { get; private set; }

    public ColorBox(Color color)
    {
        Color = color.ToString();
    }
}
```

This will act as our data. In the `MainViewModel`, add an `IList` and populate it with different colors. Here's some sample code to use if you want to copy/paste:

```csharp
public sealed class MainViewModel : ViewModel
{
    public IList Boxes { get; private set; }

    public MainViewModel()
    {
        Random RNG = new Random();
        Boxes = new List<ColorBox>(Enumerable.Range(0, 200).Select(n =>
            new ColorBox(Color.FromArgb(255, (byte) RNG.Next(255), (byte) RNG.Next(255), (byte) RNG.Next(255)))));
    }
}
```

Add an **ExportViewModel** attribute to the view model:

```csharp
[ExportViewModel("MainViewModel")]
public sealed class MainViewModel : ViewModel
{
    ...
}
```

Now, open the **MainPage.xaml** file - add a `Key` property to the Page element to locate the view model:

```xml
<Page x:Class="SampleBoxes.MainPage" 
    xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
    xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
    xmlns:local="using:SampleBoxes"
    xmlns:d="http://schemas.microsoft.com/expression/blend/2008"
    xmlns:mc="http://schemas.openxmlformats.org/markup-compatibility/2006"
    mc:Ignorable="d"
    xmlns:Mvvm="using:JulMar.Windows.Mvvm"
    Mvvm:ViewModelLocator.Key="MainViewModel">
```

Next, add a `GridView` to display the color boxes - just replace the Grid already present in the file with the following:

```xml
<Grid Background="{StaticResource ApplicationPageBackgroundThemeBrush}">
  <Grid.RowDefinitions>
    <RowDefinition Height="140"/>
    <RowDefinition/>
    <RowDefinition Height="50"/>
  </Grid.RowDefinitions>
  <Grid.ColumnDefinitions>
    <ColumnDefinition Width="120"/>
    <ColumnDefinition/>
    <ColumnDefinition Width="50"/>
  </Grid.ColumnDefinitions>
  <TextBlock Text="Colored Boxes" Style="{StaticResource PageHeaderTextStyle}" Grid.ColumnSpan="2" Grid.Column="1"/>
  <GridView Grid.Row="1" Margin="10,9,10,11" Grid.Column="1" ItemsSource="{Binding Boxes}" IsItemClickEnabled="True">
    <GridView.ItemTemplate>
      <DataTemplate>
        <Rectangle Margin="5" Fill="{Binding Color}" Width="200" Height="200"/>
      </DataTemplate>
    </GridView.ItemTemplate>
  </GridView>
</Grid>
```

Compile it and you will see design-time data - the `ViewModelLocator` works fine with the designer. It should run as well. Add a second Blank page into the project - name it **BoxDetailsPage.xaml** and use the following XAML:

```xml
<Grid Background="{StaticResource ApplicationPageBackgroundThemeBrush}">
  <Grid.RowDefinitions>
    <RowDefinition Height="140"/>
    <RowDefinition/>
    <RowDefinition Height="50"/>
  </Grid.RowDefinitions>
  <Grid.ColumnDefinitions>
    <ColumnDefinition Width="120"/>
    <ColumnDefinition/>
    <ColumnDefinition Width="50"/>
  </Grid.ColumnDefinitions>
  <Button Style="{StaticResource BackButtonStyle}" Command="{Binding GoBack}"/>
  <TextBlock Text="Box Details" Style="{StaticResource PageHeaderTextStyle}" Grid.ColumnSpan="2" Grid.Column="1"/>
  <Rectangle Fill="{Binding Color}" Width="400" Height="400" Grid.Column="1" Grid.Row="1"/>
</Grid>
```

Notice that the Back button is using a Command here to go backwards -- you could also use the LayoutAwarePage.cs here as the IPageNavigator utilizes the normal Frame navigation under the covers so you can mix and match as necessary; but here I will show you how to use a ViewModel to drive it.

Add a new ViewModel into the source called **BoxDetailsViewModel.cs** - here's the code:

```csharp
public sealed class BoxDetailsViewModel : ViewModel
{
    private string _color;

    public string Color
    {
        get
        {
            return _color;
        }
        set
        {
            SetPropertyValue(ref _color, value);
        }
    }

    [Import] public IPageNavigator PageNavigator { get; set; }

    public IDelegateCommand GoBack { get; private set; }

    public BoxDetailsViewModel()
    {
        GoBack = new DelegateCommand(() => PageNavigator.GoBack(), () => PageNavigator.CanGoBack);
    }
}
```

Notice we are importing the `IPageNavigator` rather than using the ServiceLocator. This works because the base ViewModel class makes sure to satisfy imports - if you derive from `SimpleViewModel` (which just has the **INotifyPropertyChanged** implementation) then you would need to do this step yourself. Here you can see the first usage of the page navigator to do true navigation - we are using the GoBack() method and CanGoBack property to implement our command.

Now, let's integrate this page into our system. Back in the MainViewModel, add the following code:

```csharp
[ExportViewModel("MainViewModel")]
public sealed class MainViewModel : ViewModel
{
    public IList Boxes { get; private set; }

    [Import] public IPageNavigator PageNavigator { get; set; }

    public IDelegateCommand ShowDetails { get; private set; }

    public MainViewModel()
    {
        Random RNG = new Random();
        Boxes = new List<ColorBox>(Enumerable.Range(0, 200).Select(n =>
            new ColorBox(Color.FromArgb(255, (byte) RNG.Next(255), (byte) RNG.Next(255), (byte) RNG.Next(255)))));

        ShowDetails = new DelegateCommand<ItemClickEventArgs>(OnShowDetails);
    }

    private void OnShowDetails(ItemClickEventArgs e)
    {
        ColorBox box = (ColorBox) e.ClickedItem;
        PageNavigator.NavigateTo("BoxDetailsPage", box.Color, new BoxDetailsViewModel() {Color = box.Color});
    }
}
```

Next, open the **MainPage.xaml** file and modify the `GridView` with an `EventTrigger` behavior for the `ItemClick` event to execute our new command through an InvokeCommand behavior. By default if you do not supply a CommandParameter on the InvokeCommand behavior it will pass the EventArgs (an ItemClickEventArgs in this case) which is exactly what we've coded the MainViewModel to receive on the command.

Here are the relevant changes, first add the namespace to get to the two behavior classes:

```xml
<Page x:Class="SampleBoxes.MainPage" ...
    xmlns:Interactivity="using:System.Windows.Interactivity"
    xmlns:Interactivity1="using:JulMar.Windows.Interactivity"
    Mvvm:ViewModelLocator.Key="MainViewModel">
```

Next, add the triggers collection onto the `GridView`:

```xml
<GridView Grid.Row="1" Margin="10,9,10,11" Grid.Column="1" ItemsSource="{Binding Boxes}" IsItemClickEnabled="True">
    <Interactivity:Interaction.Triggers>
        <Interactivity:EventTrigger EventName="ItemClick">
            <Interactivity1:InvokeCommand Command="{Binding ShowDetails}" />
        </Interactivity:EventTrigger>
    </Interactivity:Interaction.Triggers>
    ...
</GridView>
```

Look back at the `MainViewModel` - specifically the `ShowDetails` handler. Notice it is using the PageNavigator (imported using MEF) and navigating to the "BoxDetailsPage" which is just a string. The normal Frame navigation uses System.Type objects which ties you to the specific classes. I decided instead to use strings - which is how MVVMHelpers has always tied things together; this can be fragile if you don't manage it properly but it provides a lot more freedom when connecting views and viewmodels together and when navigating from VMs to views. The question is, "How does the navigation service know which page this corresponds to?" The answer is, a custom attribute named **ExportPage** (although you can also register it directly with the IPageNavigator service if you want to do it in code).

Open the **BoxDetailsPage.xaml.cs** file and add the following to the class declaration:

```csharp
[ExportPage("BoxDetailsPage", typeof(BoxDetailsPage))]
public sealed partial class BoxDetailsPage : Page
{
    ...
}
```

This will export the page to the navigation service and make it easily found. You can do the same for the `MainPage` and then use the page navigation service in App.xaml.cs to get to the first page if you like. If you run the app and then suspend/terminate you will see the navigation stack is preserved, but your VM is not. There are two ways to manage this issue.

1. Serialize the `DataContext` for the view
1. Add a `ViewModelLocator` onto the view to create a `BoxDetailsViewModel` (replaced when we navigate, but used during initialization from persistence) and then have the ViewModel load/save state.

I prefer the second, but the first can be useful - it has several steps to it.

1. Make the `BoxDetailsViewModel` serializable. This means adding `[DataContract]` and `[DataMamber]` attributes as well as an `OnDeserialized` handler to re-hookup the command(s) properly and recompose via MEF since the constructor isn't invoked in this case.

    ```csharp
    [DataContract]
    public sealed class BoxDetailsViewModel : ViewModel
    {
        private string _color;

        [DataMember]
        public string Color
        {
            get { return _color; }
            set { SetPropertyValue(ref _color, value); }
        }

        [Import]
        public IPageNavigator PageNavigator { get; set; }

        public IDelegateCommand GoBack { get; private set; }

        [OnDeserialized]
        public void OnDeserializing(StreamingContext context)
        {
            Initialize(); // force refresh of MEF
            GoBack = new DelegateCommand(() => PageNavigator.GoBack(), () => PageNavigator.CanGoBack);
        }

        public BoxDetailsViewModel()
        {
            GoBack = new DelegateCommand(() => PageNavigator.GoBack(), () => PageNavigator.CanGoBack);
        }
    }
    ```

1. Implement the `INavigationAware` interface on the **BoxDetailsPage.xaml.cs** implementation and save/restore the DataContext.

    ```csharp
    [ExportPage("BoxDetailsPage", typeof(BoxDetailsPage))]
    public sealed partial class BoxDetailsPage : Page, INavigationAware
    {
        public void OnNavigatedTo(NavigatedToEventArgs e)
        {
            if (DataContext == null)
            {
                if (e.State != null && e.State.ContainsKey("DataContext")) DataContext = e.State["DataContext"];
            }
        }
    
        public void OnNavigatingFrom(NavigatingFromEventArgs e)
        {
            if (e.IsSuspending)
            {
                e.State["DataContext"] = this.DataContext; // save data context
            }
        }
    }
    ```

1. Add the `BoxDetailsViewModel` as a known type in **app.xaml.cs**.

```csharp
protected async override void OnLaunched(LaunchActivatedEventArgs args)
{
    IPageNavigator pageNavigator = ServiceLocator.Instance.Resolve<IPageNavigator>();
    pageNavigator.StateManager.KnownTypes.Add(typeof(BoxDetailsViewModel));
}
```

Adding those changes will then properly save/restore the ViewModel (from the DataContext) and navigation stack on suspend/resume. As I mentioned, that's not my primary approach. Instead, I tend to export the ViewModel and wire it up with a locator in the XAML for each Page. Then I'll pass the specific page I want when I navigate to it (the last parameter is the DataContext). This is set **after** the page is created so it will replace the ViewModelLocator version during initial navigation. Then, I'll have the ViewModel persist it's properties so that when the navigation stack is restored, it will be created (by the VMLocator) and then load it's properties.

To do this, remove all the changes from above, and then wire up the ViewModelLocator to the **BoxDetaislPage.xaml**, export the `BoxDetailsViewModel` and then add **ViewModelState** attributes to each of the public properties you need to persist - in this case, just the Color property. Alternatively, you can also implement INavigationAware as we did above in the Page class and it will be called during navigation to save/restore state - this includes during suspend/resume.

So, here's the ViewModel decorated:

```csharp
[ExportViewModel("BoxDetailsVM")]
public sealed class BoxDetailsViewModel : ViewModel
{
    private string _color;

    [ViewModelState]
    public string Color
    {
        get { return _color; }
        set { SetPropertyValue(ref _color, value); }
    }

    [Import] public IPageNavigator PageNavigator { get; set; }
    public IDelegateCommand GoBack { get; private set; }

    public BoxDetailsViewModel()
    {
        GoBack = new DelegateCommand(() => PageNavigator.GoBack(), () => PageNavigator.CanGoBack);
    }
}
```

There is no need to manage serialization of this type since we aren't storing the type itself - just the color string. This will produce the same result as before.

The sample project is [here](/samples/SampleBoxes.zip).
