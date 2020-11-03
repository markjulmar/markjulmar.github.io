---
title: Synchronizing Collections in Windows Store Apps
date: 2013-05-06T00:00:00.0000000
layout: post
categories: mvvm
tags: mvvm uwp dotnet
---

One issue that often comes up when building WSAs is managing selection in your filled/snapped/full screen views.  The goal, of course, is to provide a reasonable view as the user snaps your application which is often done by providing two UIs within the page - one with a **GridView** for filled or full screen and the second with a **ListView** for snapped view.  You hide and show the two views using the Visual State Manager when using the normal templates.

If selection is provided, then ideally it would be carried over when changing views - this actually isn't hard - it can certainly be done manually and if you only allow a single selection it's dead easy even with MVVM.  Multi-selection is a bit trickier.  Enter the **SynchronizedCollectionBehavior** in MVVMHelpers.  It allows the UI to bind a collection of "selected items" to a collection in the ViewModel - so the ViewModel can track and alter the selection, but more importantly you can bind it to multiple UI things and they all track the same selection automatically.

There are two ways to activate the behavior - first is through the normal Blend framework -

```xml
<ListView ItemsSource="{Binding Names}" SelectionMode="Multiple" HorizontalAlignment="Stretch" Margin="20">
    ...
    <Interactivity:Interaction.Behaviors>
        <Interactivity1:SynchronizedCollectionBehavior Collection="{Binding SelectedNames}" />
    </Interactivity:Interaction.Behaviors>
</ListView>
```

Here the ViewModel is exposing an `ObservableCollection` of names (called SelectedNames) to manage the selected items. The actual collection (also `string` names) is just Names:

```csharp
public IReadOnlyList<string> Names { get; private set; }
public IList<string> SelectedNames { get; private set; }

public MainViewModel()
{
    Names = new List<string> {
        "Alice", "Bob", "Carol", "David", "Edgar", "Frank", "Georgia",
        "Hank", "India", "Jack", "Karen", "Larry", "Mike", "Nate", "Oscar",
        "Peter", "Quix", "Russ", "Steve", "Tonya", "Uma", "Violet", "Walter",
        "Xi", "Yvonne", "Zed"
    };

    SelectedNames = new ObservableCollection<string>();
    ...
}
```

The second activation mechanism is through a normal attached property -

```xml
<GridView ItemsSource="{Binding Names}" Grid.Column="1" Grid.Row="1" 
           SelectionMode="Extended" HorizontalAlignment="Stretch" Margin="20"
            Interactivity1:SynchronizedCollectionBehavior.IsEnabled="{Binding SelectedNames}">
```

This performs exactly the same action but allows it to be done as an attribute.

[Here is a test program](/samples/SynchronizedList.zip) which shows connecting it up to multiple `ItemsControls` and even changing the selection styles between them.
