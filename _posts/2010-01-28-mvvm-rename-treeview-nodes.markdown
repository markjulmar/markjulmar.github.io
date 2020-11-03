---
title: 'MVVM: Rename TreeView nodes'
date: 2010-01-28T00:00:00.0000000
layout: post
categories: mvvm
tags: mvvm dotnet
---

I know I said I was going to cover some services next, but I got a request last night to show how to rename TreeView nodes (ala Explorer) using the MVVM pattern in WPF.  The solution is quite easy and elegant and I thought I’d share it here.

First, the idea is we want to be able to double-click on the text portion of the `TreeViewItem` and have it allow us to type in a new name, replacing the current text.  This involves changing the visual template out (from a `TextBlock` to a `TextBox`) and then detecting that the edit is complete.

I started with the [MVVM project template](https://github.com/markjulmar/mvvmhelpers), creating the default File Explorer view.  The **TreeView** on the left is displaying folders – so I opened the **DirectoryViewModel.cs** that drives the data.

First, let’s add a property to indicate whether we are in “editing” mode or not.  This is a simple field-backed property that raises the PropertyChange notification:

```csharp
/// <summary>
/// True/False whether we are changing the name of the directory
/// </summary>
public bool IsEditingName
{
    get { return _isEditingName; }
    set { _isEditingName = value; OnPropertyChanged("IsEditingName"); }
}
```
  
I also added the private field (`_isEditingName`) into the class.  Next, locate the `Name` property that is being used to display the name.  Here we need to add a setter that renames the directory.  We want it to be robust, so we do some upfront checks and make sure to catch any I/O exceptions that occur:

```csharp
/// <summary>
/// Name of the directory
/// </summary>
public string Name
{
    get { return _data.Name; }
    // Code to rename directory
    set
    {
        string newValue = value;
        if (!string.IsNullOrEmpty(newValue))
        {
            // Remove any trailing backslash.
            string fullName = _data.FullName.TrimEnd(Path.DirectorySeparatorChar);
            // Determine the new directory name
            string directoryPath = fullName.Substring(0, fullName.Length - _data.Name.Length);
            if (!string.IsNullOrEmpty(directoryPath) && directoryPath != fullName)
            {
                string newFullName = Path.Combine(directoryPath, newValue);
                try
                {
                    _data.MoveTo(newFullName);
                }
                catch (IOException ex)
                {
                    var errorVisualizer = Resolve<IErrorVisualizer>();
                    if (errorVisualizer != null)
                    {
                        errorVisualizer.Show("Cannot rename directory", ex.Message);
                    }
                }
            }
        }
        // Tell WPF the name has changed.  Note if the same control
        // is being used to display vs. edit then the
        // binding will need to force WPF3x to re-read the property value. 
        // This is done by using a RefreshValueConverter;
        // under .NET4 this won't be necessary.
        OnPropertyChanged("Name");
        // Flip off the editing bit
        IsEditingName = false;
    }
}
```
  
Notice how changes the editing flag at the end – we assume that once the rename has occurred we are out of editing mode.  Finally, we need a way to transition from “normal” to “edit” mode and back.  In the VM these kinds of things are driven with commands – so, let’s define an `ICommand` that takes us in and out of edit mode:

```csharp
/// <summary>
/// Command used to switch to editing mode
/// </summary>
public ICommand SwitchToEditingMode { get; private set; }
```
  
And then finally initialize it in the default constructor – we also now need to chain to that constructor from the parameterized version since we always want this initialization to happen (you could also do the initialization in both, but I prefer this approach):

```csharp
/// <summary>
/// Constructor for the marker directory.  This is used to detect an expansion.
/// </summary>
private DirectoryViewModel()
{
    // Command that switches us into editing mode.
    SwitchToEditingMode = new DelegatingCommand(
                     () => IsEditingName = !IsEditingName,
                 () => _data.FullName != _data.Name);
}

/// <summary>
/// Public constructor
/// </summary>
/// <param name="di">DirectoryInfo to pull information from</param>
public DirectoryViewModel(DirectoryInfo di) : this()
{
    ...
}
```
  
That’s all the code changes we need for this – now let’s switch to the View and see how we will wire this up!  Open the **MainWindow.xaml** file and look at the **DataTemplate** used by the **TreeView**.  All of our changes will go into this template. First, we need to add a **TextBox** into the template that sits in the same space as the **TextBlock** that displays the name.  Then we need some way to switch between these two elements – I use Visibility here, you could also swap out the entire template.  We’ll use a **DataTrigger** and drive it off our new **IsEditingName** property:

```xml
<HierarchicalDataTemplate x:Key="DirectoryTemplate" ItemsSource="{Binding Subdirectories}">
     <StackPanel Orientation="Horizontal">
         <Image Width="16" Height="16"
                 Source="{Binding FullName, Converter={StaticResource iconConverter}}" />
         <Grid Margin="5,0">
                 <TextBlock x:Name="tb" Text="{Binding Name}" />
                 <!-- Editing text box -->
                 <TextBox x:Name="etb" Visibility="Collapsed" MinWidth="100"
                         Text="{Binding Name, UpdateSourceTrigger=LostFocus,
                         Converter={julmar:RefreshValueConverter}}" />
        </Grid>
    </StackPanel>
    <HierarchicalDataTemplate.Triggers>
        <DataTrigger Binding="{Binding IsSelected}" Value="True">
            <Setter TargetName="tb" Property="FontWeight" Value="Bold" />
        </DataTrigger>
        <!-- When editing mode is turned on, get rid of the TextBlock and make the TextBlock visible. -->
        <DataTrigger Binding="{Binding IsEditingName}" Value="True">
            <Setter TargetName="tb" Property="Visibility" Value="Collapsed" />
            <Setter TargetName="etb" Property="Visibility" Value="Visible" />
        </DataTrigger>
    </HierarchicalDataTemplate.Triggers>
</HierarchicalDataTemplate>
```

The other thing we need to do is set the **TextBox** binding (which also uses the **Name** property) to apply any changes when the control loses focus – that way we don’t rename as the user types!  This is done by changing the **UpdateSourceTrigger** on the binding to “**LostFocus**”. This happens to be the default setting for WPF – but not for Silverlight, so I tend to be deliberate when I want to ensure a specific behavior.  I’ve also added a converter onto the binding above – the RefreshValueConverter.  This is a no-op converter, but it forces WPF to re-read the property value after the setter is called, without it, the **TextBox** will have stale data if the rename fails.  Note that this is unnecessary in WPF4 which now always re-reads the property values automatically.  Since this targets WPF3.5, this converter will ensure proper behavior.

The final thing we need to do is somehow get in and out of editing mode.  We want this to happen when we double-click on the text element – so let’s add a JulMar behavior and action to the **TextBlock**:

```xml
<TextBlock x:Name="tb" Text="{Binding Name}">
    <Interactivity:Interaction.Triggers>
        <!-- DoubleClick activates editing mode -->
        <julmar:DoubleClickTrigger>
        <julmar:InvokeCommand Command="{Binding SwitchToEditingMode}" />
        </julmar:DoubleClickTrigger>
    </Interactivity:Interaction.Triggers>
</TextBlock>
```
  
This also requires you define the proper namespace on the Window element:

```xml
xmlns:Interactivity="clr-namespace:System.Windows.Interactivity;assembly=System.Windows.Interactivity"
```
  
I’d also like to drop _out_ of editing mode when you press ENTER in the **TextBox**. To accomplish this, let’s add an **InputBinding** to the **TextBox** for the ENTER key that invokes our **SwitchToEditingMode** command – in WPF 3.5 we need to go through a resource-based binding helper to get access to the ViewModel (which is our **DataContext**).  So, let’s defined a **BindableCommand** in the **Grid** resources (so we get the **DirectoryViewModel** as the DataContext) and then bind the input command to that:

```xml
<Grid.Resources>
    <!-- Put here so it inherits the data context properly.  We want the command to execute on directory view model -->
    <julmar:BindableCommand x:Key="EditingModeCommand"
         Command="{Binding SwitchToEditingMode}" />
</Grid.Resources>

...

<TextBox x:Name="etb" Visibility="Collapsed" MinWidth="100"
        Text="{Binding Name, UpdateSourceTrigger=LostFocus, Converter={julmar:RefreshValueConverter}}">
 <!-- Pressing ENTER in the TextBox turns off editing mode. 
 Tab or clicking away will do the same thing -->
    <TextBox.InputBindings>
        <KeyBinding Key="Enter" Command="{StaticResource EditingModeCommand}" />
    </TextBox.InputBindings>
</TextBox>
```
  
And that’s it!  If you run the app and double-click on a directory name (except the root) you can rename it!

![RenameTreeNodePic](/images/RenameTreeNodePic_thumb.jpg "RenameTreeNodePic")

Here’s the finished solution: [RenameTreeNode.zip](/samples/RenameTreeNode.zip)
