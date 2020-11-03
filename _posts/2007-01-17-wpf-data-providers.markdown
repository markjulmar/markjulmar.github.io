---
title: WPF Data Providers
date: 2007-01-17T00:00:00.0000000
layout: post
categories: wpf
tags: wpf dotnet
---

One of the nifty new features of the WPF platform is the pluggable data providers.  It ships with two out of the box:

**ObjectDataProvider:** allows you to execute binding expressions against an object and it's methods  
**XmlDataProvider:** loads an XML data source and makes it available as a binding source

Both of these derive from the abstract class **System.Data.DataSourceProvider** which implements the binding glue (**INotifyPropertyChanged**) needed for data binding.  A side note here is that you could write your own custom data provider if you needed to, although if the data is exposed through a .NET object, then the ObjectDataProvider is probably sufficient.

Using the providers is fairly easy -- let's say we have some XML data that looks like this:

```xml
<?xml version="1.0" encoding="utf-8"?>
<taxrecords>
  <entry name="Opal Harrison" state="AL" income="$51,466.81" age="27"/>
  <entry name="Eugene Black" state="FL" income="$13,314.89" age="71"/>
  <entry name="Opal Chang" state="NC" income="$225,115.15" age="41"/>
  <entry name="Gary Waters" state="WI" income="$151,788.49" age="39"/>
  <entry name="Xavier Davis" state="AK" income="$136,344.97" age="66"/>
  <entry name="Stacy Harrison" state="TX" income="$122,432.82" age="32"/>
</taxrecords>
```

The goal is to put this data into a `ListBox` - displaying the fields in the following format:

```output
Name  
State, Age, Income
```

We could clearly do all of this from procedural code -- create an `XmlReader` object, load the data and render each `XmlNode` into the listbox.  However, this is the 21st century and so we want to avoid coding as much as we can and utilize the underlying framework support instead!

We can get the data loaded into a collection source through the `XmlDataProvider`.  This is easily done in XAML:

```xml
<XmlDataProvider Source="largeXmlFile.xml" x:Key="xmlData" XPath="/taxrecords" />
```

This will look for the file "largeXmlFile.xml", create an `XmlDocument` and load the file into memory.  Notice we supply an XPath expression as part of this to indicate what we'd like the data provider to hand us as the collection itself.  In this case, we want to see everything under the node "taxrecords" which is the root of the document.  An interesting facet of this provider is that it performs it's work asynchronously -- you can see this behavior when you load very large XML files.  The UI will come up first, completely empty and then suddenly be populated with data.  The behavior can be adjusted through the **IsAsynchronous** property of the data provider.  Setting this to **false** will delay the display of the UI until the data is fully available.

Another interesting thing about this class is that we can define the XML data inline within the XAML document.  You do this with the **x:XData** tag:

```xml
<XmlDataProvider x:Key="xmlData" XPath="/taxrecords">  
    <x:Data>  
        <taxrecords>
            <entry name="Opal Harrison" state="AL" income="$51,466.81" age="27" />
            <entry name="Eugene Black" state="FL" income="$13,314.89" age="71" />  
                ...
        </taxrecords>
    </x:Data>  
</XmlDataProvider>
```

The next step is to bind this data to a ListBox control - this is a normal Data Binding expression:

```xml
<ListBox Name="lb1" Margin="10" IsSynchronizedWithCurrentItem="True"
    ItemsSource="{Binding Source={StaticResource xmlData, XPath=entry}}">
```

Notice how we use a new property of the `BindingExpression` called **XPath**.  This property allows you to identify which element(s) you want to load from the XML data source.  It is specific to this data provider and allows for any XPath expression to be supplied. This will succesfully load each of the `"/taxrecord/entry"` nodes into the ListBox, but the data itself will show up as a blank line.  This, of course, is because the data is really an XmlNode object which the ListBox has no idea how to display.  To fix this, we supply a DataTemplate to render our data: 

```xml
<ListBox.ItemTemplate>  
    <DataTemplate>  
        <StackPanel>  
            <TextBlock FontWeight="Bold" Text="{Binding XPath=@name}" />
            <StackPanel Orientation="Horizontal">  
                <TextBlock Text="{Binding XPath=@state}" />
                <TextBlock Text=", " />
                <TextBlock Text="{Binding XPath=@age}" />
                <TextBlock Text=", " />
                <TextBlock Text="{Binding XPath=@income}" />
            </StackPanel>  
        </StackPanel>  
    </DataTemplate>  
</ListBox.ItemTemplate>
```

Again, we use the `XPath` property to define what piece of information we are binding to -- attributes of our entry in this case. This template will give us the format we are looking for:  
  
![](/images/AsyncDataBind.jpg)

We can add sorting, filtering and grouping using the normal **CollectionViewSource** support.  Here is a XAML file which will present the above UI complete with sorting by the age element.  Notice how the XPath expression now moves to the CollectionView.  This is because the CollectionViewSource creates a **ListCollectionView** to manage the XML nodes because the data provider supports the **IList** interface.  The binding expression on the listbox is now simply a binding to the collection view.

```xml
<Window xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:cm="clr-namespace:System.ComponentModel;assembly=WindowsBase"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        Title="AsyncDataBind" Height="300" Width="300">
  <Window.Resources>
    <XmlDataProvider Source="largeXmlFile.xml" IsAsynchronous="True" x:Key="xmlData" XPath="/taxrecords"/>
    <CollectionViewSource x:Key="collView" Source="{Binding Source={StaticResource xmlData},XPath=entry}">
      <CollectionViewSource.SortDescriptions>
        <cm:SortDescription PropertyName="@age" Direction="Ascending"/>
      </CollectionViewSource.SortDescriptions>
    </CollectionViewSource>
  </Window.Resources>
  <Grid>
    <Grid.RowDefinitions>
      <RowDefinition Height="*"/>
      <RowDefinition Height="Auto"/>
    </Grid.RowDefinitions>
    <ListBox Name="lb1" Margin="10" IsSynchronizedWithCurrentItem="True" ItemsSource="{Binding Source={StaticResource collView}}">
      <ListBox.ItemTemplate>
        <DataTemplate>
          <StackPanel>
            <TextBlock FontWeight="Bold" Text="{Binding XPath=@name}"/>
            <StackPanel Orientation="Horizontal">
              <TextBlock Text="{Binding XPath=@state}"/>
              <TextBlock Text=", "/>
              <TextBlock Text="{Binding XPath=@age}"/>
              <TextBlock Text=", "/>
              <TextBlock Text="{Binding XPath=@income}"/>
            </StackPanel>
          </StackPanel>
        </DataTemplate>
      </ListBox.ItemTemplate>
    </ListBox>
    <Button Grid.Row="1">
      <StackPanel Orientation="Horizontal">
        <TextBlock Text="{Binding ElementName=lb1, Path=Items.Count}"/>
        <TextBlock Text=" Items"/>
      </StackPanel>
    </Button>
  </Grid>
</Window>
```

Data binding in WPF is extremely powerful -- I am constantly amazed at how much procedural code you can dump in favor of markup with creative bindings.  In the next post I'll talk a bit more about asynchronous bindings outside of the two data providers.

Until then..
