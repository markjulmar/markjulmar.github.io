---
title: Menus and MVVM (2)
date: 2009-04-29T00:00:00.0000000
layout: post
categories: mvvm
tags: mvvm dotnet
---

In the previous post, I wrote about using the MVVM pattern with Context Menus, but you can use it with top-level menus too -- for example, perhaps I'd like to manage a set of options through a set of checkboxes.  First, let's start with my list of options - for simplicity, we'll just make it an enumeration:

```csharp
public enum Test  
{  
   A,  
   B,  
   C,  
   D,  
   E,  
   F  
}
```

Next, let's define a ViewModel to wrap this - each entry should support a single value and a boolean binding to determine whether it is checked:

```csharp
public class CheckValues  
{  
    public Test Value { get; set; }  
    public string Text { get { return Value.ToString(); } }  
    public bool IsChecked { get; set; }  
}
```

Very likely this object would also implement INotifyPropertyChanged and fire change notifications, in this case it's not necessary since we aren't changing the values from code behind.  The next step is to expose this as a bindable collection - since it's not changing we'll just use a `List<T>`.

```csharp
public List<CheckValues> EnumValues { get; set; }  

...

EnumValues = new List<CheckValues>();  
foreach (object t in Enum.GetValues(typeof (Test)))  
{  
   EnumValues.Add(new CheckValues {Value = (Test) t});  
}
```

Finally, in the XAML itself we want to databind to the collection of CheckValues, supplying the appropriate support for each MenuItem to bind the menu checks to the collection:

```xml
<Menu DockPanel.Dock="Top">  
   <MenuItem Header="Enum Values" ItemsSource="{Binding EnumValues}">  
      <MenuItem.ItemContainerStyle>  
         <Style TargetType="MenuItem">  
            <Setter Property="Header" Value="{Binding Text}" />  
            <Setter Property="IsCheckable" Value="True" />  
            <Setter Property="IsChecked" Value="{Binding IsChecked}" />  
         </Style>  
      </MenuItem.ItemContainerStyle>  
   </MenuItem>  
</Menu>
```

That gives us our nice "checkable" menu - when the user clicks a checkmark, it sets the appropriate boolean in the code behind which could trigger logic, set values, etc.

But wait!  Often with checkboxes you want mutual exclusive choices - like the RadioButton groups.  This was a feature of Windows Forms which unfortunately did not make it into WPF, but we can make it work that way.  We could use the above logic and have the selection of a specific CheckValue unselect any others, or we could take advantage of RadioButtons to do this -- unfortunately I've not found a way that doesn't invoice a little code behind so it's not quite as clean as I'd like it to be, but it is possible.

First, let's define an exclusive ViewModel object, here we will track the "current" selection:

```csharp
public class ExclusiveCheckValues  
{  
    public static Test SelectedValue { get; set; }  
    public Test Value { get; set; }  
  
    public string Text { get { return Value.ToString(); } }  
    public bool IsChecked  
    {  
        get { return Value == SelectedValue; }  
        set  
        {  
            if (value)  
                SelectedValue = Value;  
        }  
    }  
}  
```

Next, we populate it into a bindable collection just as before and then we create some slightly more complex XAML for our menu items.  The trick here is to put a RadioButton into the menu item and associate it with a group.  I have to give a shout out to Dr. WPF from the wpf-disciples list some kudos here - he posted this trick on the MSDN forums ([http://social.msdn.microsoft.com/Forums/en-US/wpf/thread/76c14163-9462-44d6-b0eb-6f22a006b423](http://social.msdn.microsoft.com/Forums/en-US/wpf/thread/76c14163-9462-44d6-b0eb-6f22a006b423)).  It turns out the MenuItem.Icon is positioned just where the normal check mark would be so if we place a styled RadioButton as the MenuItem.Icon we can make it look just like the checkmark:

```xml
<Menu DockPanel.Dock="Top">  
  
   <Menu.Resources>  
      <RadioButton x:Key="rb" x:Shared="false" HorizontalAlignment="Left"  
                   GroupName="MenuGroup" IsChecked="{Binding IsChecked}">  
         <RadioButton.Template>  
            <ControlTemplate TargetType="{x:Type RadioButton}">  
               <Path x:Name="check" Margin="7,0,0,0" Visibility="Collapsed" VerticalAlignment="Center"  
                      Fill="{TemplateBinding MenuItem.Foreground}" FlowDirection="LeftToRight"  
                      Data="M 0,5.1 L 1.7,5.2 L 3.4,7.1 L 8,0.4 L 9.2,0 L 3.3,10.8 Z"/>  
               <ControlTemplate.Triggers>  
                   <Trigger Property="IsChecked" Value="True">  
                      <Setter TargetName="check" Property="Visibility" Value="Visible" />  
                   </Trigger>  
                </ControlTemplate.Triggers>  
             </ControlTemplate>  
          </RadioButton.Template>  
       </RadioButton>  
    </Menu.Resources>  
  
    <MenuItem Header="Single Select Enum Values" ItemsSource="{Binding ExclusiveEnumValues}">  
       <MenuItem.ItemContainerStyle>  
          <Style TargetType="MenuItem">  
             <Setter Property="Header" Value="{Binding Text}" />  
             <Setter Property="Icon" Value="{DynamicResource rb}" />  
             <EventSetter Event="Click" Handler="OnMenuItemClick" />  
          </Style>  
       </MenuItem.ItemContainerStyle>  
    </MenuItem>  
</Menu>
```

Here, we've placed the RadioButton into our resources - but marked it as non-shareable.  The issue is that Styles always share property setters - and if you tried to do this:

```xml
<Style TargetType="MenuItem">  
    <Setter Property="Icon">  
        <Setter.Value>  
            <RadioButton ... />  
        </Setter.Value>  
    </Setter>  
    ...
</Style>
```

You will get a very strange error message: "Cannot assigned type 'RadioButton' to type 'object'".  Which makes absolutely no sense until you realize the XAML parser is trying to assign the **same** RadioButton to every MenuItem.Icon.  The non-shareable resource fixes that issue.  The last step is to set the check -- this is done through the EventSetter in the above template.  It's wired to the little bit of code behind:

```csharp
private void OnMenuItemClick(object sender, RoutedEventArgs e)  
{  
   ((RadioButton) ((MenuItem) sender).Icon).IsChecked = true;  
}
```

This then sets the `IsChecked` property which will, in turn, select that item.  Now you can look at `ExclusiveCheckValue.SelectedValue` to get the current item.  This isn't the most elegant solution, nor is it the only solution but it's an interesting one to think on.  If I were truly building this into a production application, I'd probably look at controlling the selection in the ModelView (i.e. enumerate through my other choices and deliberately turn them off).  But this does illustrate how powerful WPF is!

[Here's the example project with both styles.](/samples/menucheckbox.zip)
