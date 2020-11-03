---
title: Managing focus with MVVM in Windows Store Apps
date: 2013-11-26T00:00:00.0000000
layout: post
categories: mvvm
tags: mvvm uwp dotnet
---

One thing that is always troublesome in MVVM is focus-management. How can we push focus to a specific control during business logic processing and still keep a clean separation of UI and logic? There are likely several good solutions out there, but one I have used in the past is a Blend Behavior called [SetFocusAction](https://github.com/markjulmar/mvvmhelpers/blob/04e100cebc75b9b2c4c72af26bfe8dc7b683dd83/Julmar.Wpf.Helpers/Julmar.Wpf.Behaviors/Actions/SetFocusAction.cs). I ported this behavior to the 8.1 Windows Store model (and it also exists in 8.0 as part of my full behaviors package with [MVVMHelpers](https://github.com/markjulmar/mvvmhelpers/blob/04e100cebc75b9b2c4c72af26bfe8dc7b683dd83/Metro81/MVVMHelpers.Metro/Interactivity/SetFocusAction.cs)).

Using it is pretty easy - let's pretend we are entering tags for a blog post or some data management system. We can type in the tag and click a button - but we want focus to always return to the TextBox where we enter the tag name once it's added to the list. We also want focus to initially start in the TextBox. Our UI will look something like this:

![](/images/FocusApp.jpg "FocusApp")

To generate this UI, we'll start with a Blank application for Windows 8.1 and then add a reference to MVVMHelpers - the easiest way to get it is through NuGet.  Just search for "MVVMHelpers" and then select the version for Windows 8.1.

![](/images/MVVMHelpers81.jpg "MVVMHelpers81")

> **Note**:
>
> I could not find a way to distinguish between Windows 8.0 and 8.1 in a Nuget package; so I ended up generating a separate package for Win8.1 so I can run some powershell scripts and include different DLL dependencies.  If anyone knows how to use a single package to multi-target .NET 4.5 with either Win8 or Win8.1 I'd love to know the secret sauce!

Next, let's generate the UI - here's some simple XAML to create a TextBox, button and ListView which we'll populate with data and some behaviors:

```xml
<Page xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation" xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml" xmlns:d="http://schemas.microsoft.com/expression/blend/2008" xmlns:mc="http://schemas.openxmlformats.org/markup-compatibility/2006" xmlns:focusTests="using:FocusTests" x:Class="FocusTests.MainPage" mc:Ignorable="d">
  <Grid Background="{ThemeResource ApplicationPageBackgroundThemeBrush}">
    <Grid.RowDefinitions>
      <RowDefinition Height="100"/>
      <RowDefinition/>
    </Grid.RowDefinitions>
    <Grid.ColumnDefinitions>
      <ColumnDefinition Width="100"/>
      <ColumnDefinition/>
    </Grid.ColumnDefinitions>
    <TextBlock Text="Focus Test" Grid.Column="1" VerticalAlignment="Bottom" FontSize="36" Foreground="{StaticResource ApplicationHeaderForegroundThemeBrush}"/>
    <Grid Grid.Row="1" Grid.Column="1">
      <Grid.RowDefinitions>
        <RowDefinition Height="Auto"/>
        <RowDefinition/>
      </Grid.RowDefinitions>
      <Grid>
        <Grid.ColumnDefinitions>
          <ColumnDefinition/>
          <ColumnDefinition Width="Auto"/>
        </Grid.ColumnDefinitions>
        <TextBox x:Name="EntryLabelTextBox" PlaceholderText="New Label" Margin="5"/>
        <Button Content="Add" Padding="20,5" Margin="5" Grid.Column="1"/>
      </Grid>
      <ListView Grid.Row="1" Margin="5">
        <ListView.ItemsPanel>
          <ItemsPanelTemplate>
            <ItemsWrapGrid/>
          </ItemsPanelTemplate>
        </ListView.ItemsPanel>
        <ListView.ItemTemplate>
          <DataTemplate>
            <Border CornerRadius="10" Background="SkyBlue" BorderBrush="DeepSkyBlue" MinWidth="75">
              <TextBlock HorizontalAlignment="Center" Foreground="DarkBlue" VerticalAlignment="Center" Text="{Binding}" Margin="10"/>
            </Border>
          </DataTemplate>
        </ListView.ItemTemplate>
      </ListView>
    </Grid>
  </Grid>
</Page>
```

Next, let's add a View Model to the project - derive it from `SimpleViewModel` in the MVVMHelpers framework - this is the simplest of the view model base classes and provides just the basic `INotifyPropertyChanged` implementation. We'll need several things:

1. A command to handle the button click - let's call it AddTag
1. An `IList` collection to manage the tags - we'll call that Tags
1. An event to move focus to our TextBox - call that AddComplete and make it an `Action` event.

The implementation of the command should just add a string to the list - I am going to pass the string along with the command, just to show how to do that, but you can also just bind the TextBox.Text property to a string (which I will also do here to show how that might be used). In the handler, add the string, clear the string property (which will clear the TextBox) and then raise the event.

Here's my implementation:

```csharp
using System;
using System.Collections.Generic;
using System.Collections.ObjectModel;

using JulMar.Windows.Interfaces;
using JulMar.Windows.Mvvm;

namespace FocusTests
{
    [ExportViewModel("MainVM")]
    public sealed class MainViewModel: SimpleViewModel
    {
        private string _text;
        public string Text {
            get {
                return this._text;
            }
            set {
                SetPropertyValue(ref _text, value);
            }
        }

        public event Action AddComplete = delegate {};
        public IDelegateCommand AddTag {
            get;
            private set;
        }
        public IList < string > Tags {
            get;
            private set;
        }

        public MainViewModel() {
            AddTag = new DelegateCommand < string > (OnAddTag, s = >!string.IsNullOrEmpty(s) && !Tags.Contains(s));
            Tags = new ObservableCollection < string > ();
        }

        private void OnAddTag(string newTag) {
            Tags.Add(newTag);
            Text = string.Empty;
            AddComplete();
        }
    }
}
```

Finally, let's wire the ViewModel into the system - set an instance of the `MainViewModel` as the `DataContext` using the ViewModelLocator.Key attached property in XAML. This requires the view model be exported (that's what the attribute applied to the ViewModel class does above).

```xml
<Page xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
    xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
    xmlns:d="http://schemas.microsoft.com/expression/blend/2008"
    xmlns:mc="http://schemas.openxmlformats.org/markup-compatibility/2006"
    xmlns:interactivity="using:Microsoft.Xaml.Interactivity"
    xmlns:core="using:Microsoft.Xaml.Interactions.Core"
    xmlns:julmar="using:JulMar.Windows.Interactivity"
    xmlns:focusTests="using:FocusTests"
    xmlns:mvvm="using:JulMar.Windows.Mvvm"
    x:Class="FocusTests.MainPage"
    mvvm:ViewModelLocator.Key="MainVM"
    d:DataContext="{d:DesignInstance Type=focusTests:MainViewModel, IsDesignTimeCreatable=True}"
    mc:Ignorable="d"/>
```

Next, let's shift focus initially to the TextBox using an `EventTriggerBehavior` from the Blend SDK, and the `SetFocusAction` from the JulMar MVVMHelpers library. You can do this in Blend (drag / drop), or by hand if you add the required namespaces to your XAML (they are included above as `interactivity`, `core`, and `julmar`.

Locate the TextBox and add the following XAML to hook into the `Loaded` event and push focus into the TextBox. Unfortunately, you must supply a Target as the Action elements in this framework do not have an associated XAML element and there is no way to pick that up in code.

```xml
<TextBox x:Name="EntryLabelTextBox" PlaceholderText="New Label" Margin="5" Text="{Binding Text, Mode=TwoWay}">
  <interactivity:Interaction.Behaviors>
    <core:EventTriggerBehavior EventName="Loaded">
      <julmar:SetFocusAction Target="{Binding ElementName=EntryLabelTextBox}"/>
    </core:EventTriggerBehavior>
  </interactivity:Interaction.Behaviors>
</TextBox>
```

Next, add an `EventTriggerBehavior` to the Button to invoke the AddTag command when it is clicked - pass the text as the parameter. I am going to use the MVVMHelpers version as it is slightly more flexible.

```xml
<Button Content="Add" Padding="20,5" Margin="5" Grid.Column="1">
  <interactivity:Interaction.Behaviors>
    <core:EventTriggerBehavior EventName="Click">
      <julmar:InvokeCommand Command="{Binding AddTag}" CommandParameter="{Binding ElementName=EntryLabelTextBox, Path=Text}"/>
    </core:EventTriggerBehavior>
  </interactivity:Interaction.Behaviors>
</Button>
```

Databind the `ListView` so it uses your Tags collection, and the TextBox.Text property to your Text property in the ViewModel - this will allow us to manipulate the data presented in both from the ViewModel.

Finally, add a `ViewModelTriggerBehavior` to the root Grid element - hook it into your AddComplete event and use the same `SetFocusAction` you added to the TextBox initially to move focus into it.

```xml
<interactivity:Interaction.Behaviors>
  <julmar:ViewModelTriggerBehavior EventName="AddComplete" Target="{Binding}">
    <julmar:SetFocusAction Target="{Binding ElementName=EntryLabelTextBox}"/>
  </julmar:ViewModelTriggerBehavior>
</interactivity:Interaction.Behaviors>
```

Run the app - it should push focus initially into the TextBox. When you click the button (with touch, mouse or keyboard), it will add the tag and then _move focus back into the TextBox_.

Here's the final solution: [FocusTests.zip](/samples/WSA.FocusTests.zip)
