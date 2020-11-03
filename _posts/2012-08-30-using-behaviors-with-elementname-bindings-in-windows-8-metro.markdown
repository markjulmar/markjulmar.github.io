---
title: Using behaviors with ElementName bindings in Windows 8 Metro
date: 2012-08-30T00:00:00.0000000
layout: post
categories: mvvm
tags: mvvm dotnet
---

I got an email yesterday from a developer who was trying to use MVVMHelpers and the new **System.Windows.Interactivity** support for Windows 8 and was having trouble. Specifically, he was trying to do this:

```xml
<ListView x:Name="lstModules" Padding="10,0,0,60" IsItemClickEnabled="true" ItemsSource="{Binding Modules}">
  <interactivity:Interaction.Triggers>
    <interactivity:EventTrigger EventName="ItemClick">
      <julmar:InvokeCommand Command="{Binding SelectModule}" CommandParameter="{Binding ElementName=lstModules,Path=SelectedItem}"/>
    </interactivity:EventTrigger>
  </interactivity:Interaction.Triggers>
</ListView>
```

The idea was to have a command get invoked when an item is tapped in the ListView - passing the selected item as the parameter to the command. He was always getting **null** in his command handler. There are actually two issues with this snippet, one is in the setup of the **ListView** itself and the second is a restriction of attached properties today in Windows 8 Metro. Let's tackle them in the opposite order.

### ElementName bindings and Namescopes

Windows 8 XAML uses the same principles for XAML parsing that WPF and Silverlight do - a single XAML file encompasses a _name scope_ which is used to find elements by name in that area.  Within a template (data or control), the name scope is limited to that specific template instantiation.  The behaviors are being attached via the **Interaction.Triggers** collection - and, while they are **FrameworkElement** classes so they inherit the DataContext, they are not part of the visual tree and so do not get automatic access to the XAML name scope. A few of the behaviors walk the visual tree and hook into the namescope to look up things (specifically targeted actions), but most of them remain unaware of it.  Now, what that means practically, is that **{Binding ElementName}** style bindings as shown above _will not work inside the trigger or behaviors collections_.

Most of the time, I'm actually ok with this restriction - I don't really want to pass much of the UI shell into my ViewModel, they are intentionally separated.  But it can be convenient sometimes, for triggers and such - so how do we manage that?

#### Enter: **NameScopeBinding**

In MVVMHelpers 1.02, I added a new type which solves this problem by using resources.  Essentially, you can add a **NameScopeBinding** to your resources, bind it to the element in question and then use **{StaticResource}** in your behavior/trigger to get access to that element - kind of an indirect lookup like a stored field.  Here's an example:

```xml
<Page.Resources>
    <Behaviors:NameScopeBinding x:Key="ItemListView" Element="{Binding ElementName=lstModules}" />
</Page.Resources>
    ...
<julmar:InvokeCommand Command="{Binding SelectModule}"
    CommandParameter="{Binding Element.SelectedItem, Source={StaticResource ItemListView}}">
```

Notice how the **CommandParameter** locates the namescope binder and then grabs the element and selected item - this works and solves at least this part of the problem. You could use this approach to get access to any named element in a place where it's not normally available to you.

### ListViews in Windows 8

The second issue is how the ListView actually works - notice in the original XAML block, it has the **IsItemClickEnabled** property set? This actually tells the ListView to work as a touch-style control, i.e. you tap items and they raise an ItemClick event, _but do not actually perform a selection!_ This means the approach of passing a **SelectedItem** is actually flawed because the ListView won't track selection here. So, if you _really_ want to use this approach, you need to turn that feature off, allow selection and wire up to the **SelectionChanged** event instead (or more ideally, data bind the **SelectedItem** property of the ListView to some property in the ViewModel then you don't need the parameter at all.) Another approach is to take advantage of the **ItemClick** event - it passes an **ItemClickEventArgs** which includes the tapped item. MVVMHelpers and **InvokeCommand** will help you here - if you do not wire up the **CommandParameter** (or it's null) then it will automatically pass the trigger parameter - which in this case would be the **ItemClickEventArgs**. You could then grab the clicked item from that in your ViewModel. So, if you just did this:

```xml
<interactivity:Interaction.Triggers>
    <interactivity:EventTrigger EventName="ItemClick">
        <julmar:InvokeCommand Command="{Binding SelectModule}" />
    </interactivity:EventTrigger>
</interactivity:Interaction.Triggers>
```

You'd actually get the **EventArgs** for the **ItemClick** passed to you. So you could do this in the ViewModel:

```csharp
SelectModule = new DelegateCommand<ItemClickEventArgs>(OnSelectModuleFromItemClick);
...

private void OnSelectModuleFromItemClick(ItemClickEventArgs e)
{
    DoSelectModule(e.ClickedItem as Module);
}
```

As I mentioned, I'm not a big fan of this approach because it's leaking some UI implementation details into the ViewModel and I'm a fan of keeping them separate, however I also live in the real world and sometimes you have to break rules :-)

Thanks to Frank Nguyen for reporting it to me and working through it!

Happy coding!
