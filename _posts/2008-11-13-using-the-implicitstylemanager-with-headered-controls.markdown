---
title: Using the ImplicitStyleManager with Headered controls
date: 2008-11-13T00:00:00.0000000
layout: post
categories: mvvm
tags: mvvm dotnet
---

I've been playing with the Silverlight toolkit (released at PDC) this week and ran across a pretty nasty edge case related to `HeaderedContentControl` and the implicit style manager.  If you create something like this, where the Header is set to a Visual of some kind:

```xml
<UserControl x:Class="SilverlightPrototype.Page"  
    xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"  
    xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"  
    xmlns:sys="clr-namespace:System;assembly=mscorlib"  
    xmlns:Controls="clr-namespace:Microsoft.Windows.Controls;assembly=Microsoft.Windows.Controls">  
  
    <StackPanel x:Name="LayoutRoot" Background="White">  
       <Button Click="Button_Click" Content="Change Style" />  
       <Controls:HeaderedContentControl>  
     <Controls:HeaderedContentControl.Header>  
             <TextBlock Text="Header" Foreground="DarkBlue" FontWeight="Bold" />  
          </Controls:HeaderedContentControl.Header>      <TextBlock Text="Body" />  
       </Controls:HeaderedContentControl>  
    </StackPanel>  
    </UserControl>
```

And then you change the style of the headered control when the button is clicked:

```csharp
private void Button_Click(object sender, RoutedEventArgs e)  
{
    Uri uri = new Uri(@"SilverlightPrototype;component/theming/simplestyle.xaml", UriKind.Relative);  
    ImplicitStyleManager.SetResourceDictionaryUri(LayoutRoot, uri);  
    ImplicitStyleManager.SetApplyMode(LayoutRoot, ImplicitStylesApplyMode.OneTime);  
    ImplicitStyleManager.Apply(LayoutRoot);  
}
```

Where the style is just a simple ContentPresenter or ContentControl for the header and the body:

```xml
<Style TargetType="Controls:HeaderedContentControl">  
   <Setter Property="Template">  
      <Setter.Value>  
        <ControlTemplate TargetType="Controls:HeaderedContentControl">  
           <StackPanel Orientation="Horizontal">  
              <ContentPresenter Content="{TemplateBinding Header}" ContentTemplate="{TemplateBinding HeaderTemplate}" Margin="10" />  
              <ContentPresenter Content="{TemplateBinding Content}" ContentTemplate="{TemplateBinding ContentTemplate}" Margin="10" />  
           </StackPanel>  
        </ControlTemplate>  
     </Setter.Value>  
   </Setter>  
</Style>  
```

Silverlight will crash deep in the **DependencyProperty.SetValue** code with an **ArgumentException** indicating that the TextBlock for the header is invalid.  The issue appears to be that the TextBlock defined as the header content is already part of the visual tree and therefore cannot be bound to the new control template.  Content works fine so there is some other path being taken for that property.

The workaround is pretty easy, just use a non-visual in the header and then supply a DataTemplate to give it the appropriate visual tree.  Silverlight will re-create the visual tree each time from the data template so we don't have the reuse issue:

```xml
<StackPanel x:Name="LayoutRoot" Background="White">  
    <StackPanel.Resources>  
       <!-- Header works as long as template is supplied - i.e. non-visual element applied directly to header -->  
       <DataTemplate x:Key="HeaderTemplate">  
          <TextBlock Foreground="DarkBlue" FontWeight="Bold" Text="{Binding}" />  
       </DataTemplate>  
    </StackPanel.Resources>  

    <Button Click="Button_Click" Content="Change Style" />  

    <Controls:HeaderedContentControl HeaderTemplate="{StaticResource HeaderTemplate}">  
     <Controls:HeaderedContentControl.Header>  
          <sys:String>Header Text</sys:String>  
       </Controls:HeaderedContentControl.Header>  
       <TextBlock Text="Body" />  
    </Controls:HeaderedContentControl>  
</StackPanel>  
```

If you need more than a string, then just define an object and assign that instead - for example:

```csharp
class HeaderStuff  
{  
    public string Text { get; set; }  
    public bool IsChecked { get; set; }  
}
```

```xml
<DataTemplate x:Key="HeaderStuff">  
   <StackPanel>  
      <Rectangle Fill="Blue" Stroke="Yellow" Width="50" ... />  
      <CheckBox IsChecked="{Binding IsChecked}" Content="{Binding Text}" />  
   </StackPanel>  
</DataTemplate>  
  
...  
  
<Controls:HeaderedContentControl HeaderTemplate="{StaticResource HeaderStuff}">  
   <Controls:HeaderedContentControl.Header>  
      <me:HeaderStuff IsChecked="True" Text="This is a checkbox" />  
   </Controls:HeaderedContentControl.Header>  
</Controls:HeaderedContentControl.Header>
```
