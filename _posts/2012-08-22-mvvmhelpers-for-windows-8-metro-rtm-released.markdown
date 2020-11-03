---
title: MVVMHelpers for Windows 8 Metro RTM released
date: 2012-08-22T00:00:00.0000000
layout: post
categories: mvvm
tags: mvvm dotnet
---

I have updated the MVVM Helpers for Metro to the RTM Windows 8 build. If you've been following the source changes, you might notice I've included full support for Blend Behaviors - even though Blend itself doesn't support them!  To accomplish this, I ported the **System.Windows.Interactivity.dll** to Metro and include it as part of the package - I'll remove this once Blend supports behaviors officially.

I've also ported the most useful of the Blend behaviors and behaviors from the WPF MVVMHelpers.  That means you can do things like this:

```xml
<Button Content="Call JustShowText" Margin="20" Padding="10,5">
  <Interactivity:Interaction.Triggers>
    <Interactivity:EventTrigger EventName="Click">
      <Actions:CallMethodAction TargetObject="{Binding}" MethodName="JustShowText"/>
    </Interactivity:EventTrigger>
  </Interactivity:Interaction.Triggers>
</Button>
```

This will bind to the active data context (which is inherited from the owner where the behavior is applied) and call the "JustShowText" method. It supports public methods with no parameters and a void return type, or a public method with void return, taking two parameters - the sender and parameter from the trigger (often an **EventArgs** or **RoutedEventArgs**, remember that **RoutedEventArgs** no longer derives from **EventArgs** so they are distinct). It will locate the latter if it can and fallback to the no-parameter version if it can't locate a matching signature. For example, both of these methods would be valid:

```csharp
public void JustShowText() { Text = "Hello from JustShowText"; }

// CallMethodAction prefers this form - so if this is present
// then this will be the called version.
public void JustShowText(object sender, object e)
{
    Text = sender.GetType() + " : " + e.GetType();
}
```

You can also bind to **ICommand** implementations:

```xml
<Rectangle Fill="{Binding Color}" Margin="50" Stroke="White" StrokeThickness="1" Width="200" Height="200">
  <Interactivity:Interaction.Triggers>
    <Interactivity:EventTrigger EventName="PointerEntered">
      <Interactivity1:InvokeCommand Command="{Binding MouseEnter}"/>
    </Interactivity:EventTrigger>
    <Interactivity:EventTrigger EventName="PointerExited">
      <Interactivity1:InvokeCommand Command="{Binding MouseLeave}"/>
    </Interactivity:EventTrigger>
  </Interactivity:Interaction.Triggers>
</Rectangle>
```

I've also ported many of the popular actions, **CallMethod**, **InvokeCommand**, **ChangeProperty**, **GotoState** and **PlaySound** - for example, you can change property values based on double-click events, in this next example, the behavior will animate the color change for the brush (stored in resources):

```xml
<Rectangle Margin="50" Stroke="White" StrokeThickness="1" Width="200" Height="200" HorizontalAlignment="Left" VerticalAlignment="Top" Fill="{StaticResource rcBrush}">
  <Interactivity:Interaction.Triggers>
    <Interactivity1:DoubleClickTrigger>
      <Interactivity1:ChangePropertyAction TargetObject="{StaticResource rcBrush}" PropertyName="Color" Duration="0:0:3">
        <Interactivity1:ChangePropertyAction.Value>
          <Color>Blue</Color>
        </Interactivity1:ChangePropertyAction.Value>
      </Interactivity1:ChangePropertyAction>
    </Interactivity1:DoubleClickTrigger>
  </Interactivity:Interaction.Triggers>
</Rectangle>
```

Here we will watch for the "B" key and change the stroke and thickness for this shape. I've also applied a **DragPositionBehavior** which allows the pointer (mouse/touch/stylus) to move the element around:

```xml
<Rectangle x:Name="RedRect" Fill="Red" Margin="50" Stroke="White" StrokeThickness="1" Width="200" Height="200" HorizontalAlignment="Left" VerticalAlignment="Bottom">
  <Interactivity:Interaction.Behaviors>
    <Interactivity1:DragPositionBehavior/>
  </Interactivity:Interaction.Behaviors>
  <Interactivity:Interaction.Triggers>
    <Input:KeyTrigger Key="B">
      <Interactivity1:ChangePropertyAction PropertyName="Stroke" Value="Gold" TargetName="RedRect"/>
      <Interactivity1:ChangePropertyAction PropertyName="StrokeThickness" Value="3" TargetName="RedRect"/>
    </Input:KeyTrigger>
  </Interactivity:Interaction.Triggers>
</Rectangle>
```

Finally, here's an example which data binds to a boolean in the ViewModel and changes the property based on the state of that value. The ViewModel property would be a normal **INotifyPropertyChanged** implementation (**BindableBase**, or the MVVMHelpers **SimpleViewModel**/**ViewModel**):

```xml
<Rectangle x:Name="GreenRect" Fill="Green" Margin="50" Stroke="White" StrokeThickness="1" Width="200" Height="200" HorizontalAlignment="Right" VerticalAlignment="Bottom">
  <Interactivity:Interaction.Triggers>
    <Interactivity1:DataTrigger Binding="{Binding ShowAdvanced}" Comparison="Equal" Value="True">
      <Interactivity1:ChangePropertyAction PropertyName="Fill">
        <Interactivity1:ChangePropertyAction.Value>
          <SolidColorBrush Color="Orange"/>
        </Interactivity1:ChangePropertyAction.Value>
      </Interactivity1:ChangePropertyAction>
    </Interactivity1:DataTrigger>
  </Interactivity:Interaction.Triggers>
</Rectangle>
```

You can get the behaviors and MVVM support from Nuget, just search for **MVVMHelpers-Metro** (MVVMHelpers will find it too, but you'll also find the WPF version). Of course, all the source code is up on GitHub!
