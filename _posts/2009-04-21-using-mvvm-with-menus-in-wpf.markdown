---
title: Using MVVM with Menus in WPF
date: 2009-04-21T00:00:00.0000000
layout: post
categories: mvvm
tags: mvvm wpf dotnet
---

One question I've fielded a couple of times is how to manage menus, primarily context menus, with the MVVM pattern.  It turns out to be pretty easy once you know the "trick".  The key thing to keep in mind is that menus are just **ItemsControls** - they support data templating and binding like any other ItemsControl.  However, the part where people get lost is in hooking up the commands.  Here's the trick:

**Step 1:** Define some code behind construct to manage each menu item in a hierarchial fashion.  Here's one I've used:

```csharp
public class MenuItem  
{  
   public string Text { get; set; }  
   public List<MenuItem> Children { get; private set; }  
   public ICommand Command { get; set; }  

   public MenuItem(string item)  
   {  
       Text = item;  
       Children = new List<MenuItem>();  
   }  
}
```

**Step 2:** Create your menu structure using the above container.

This involves just creating a `List<MenuItem>` which holds the root nodes and using a **DelegatingCommand** to wire up some code behind method.  Here's an example:

```csharp
public List<MenuItem> MenuOptions
{
    get
    {
        var menu = new List<MenuItem>();
        if (SupportedFileFormats.Count > 0)
        {
            var mi = new MenuItem("O_pen");
            foreach (var fl in SupportedFileFormats)
            {
                var sff = fl;
                mi.Children.Add(new MenuItem(fl.Attributes.Description)
                    {Command = new DelegatingCommand(() => { LoadFromFormat(sff); })});
            }

            menu.Add(mi);
        }

        menu.Add(new MenuItem("Close _All")
            {Command = new DelegatingCommand(OnCloseAll, () => FileList.Count > 0)});
        return menu;
    }
}
```

Note the use of an anonymous method to suck the correct file format into the command handler.  A nice trick so the command is executed with some contextual information of what you clicked on.

Finally, you need to setup the context menu style:

```xml
<Style x:Key="ContextMenuItemStyle">
    <Setter Property="MenuItem.Header" Value="{Binding Text}"/>
    <Setter Property="MenuItem.ItemsSource" Value="{Binding Children}"/>
    <Setter Property="MenuItem.Command" Value="{Binding Command}" />  
</Style>
```

And then use the style where you want the menu to appear:

```xml
<StackPanel Orientation="Horizontal">
    <Image Source="{Binding Image}" Width="16" Height="16" />
    <TextBlock Margin="5" HorizontalAlignment="Left" VerticalAlignment="Center" Text="{Binding Header}" />
    <StackPanel.ContextMenu>
        <ContextMenu ItemContainerStyle="{StaticResource ContextMenuItemStyle}" ItemsSource="{Binding MenuOptions}" />  
    </StackPanel.ContextMenu>  
</StackPanel>
```

Now the context menu is populated and properly dispatches to your ViewModel commands!
