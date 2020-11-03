---
title: Adding a watermark to a PasswordBox in a Windows Store app
date: 2013-05-28T00:00:00.0000000
layout: post
categories: mvvm
tags: mvvm uwp dotnet
---

In the previous post, I wrote about a Blend behavior for Windows Store apps to add a watermark to a **TextBox**.  The next question I got was "Well, what about a **PasswordBox**?" **PasswordBox** is a bit tricker since it doesn't allow text to be displayed in the clear - so our little trick of changing the **Text** property doesn't work here.  So, instead, let's get a little hacky (or clever depending on how you look at it I suppose).  We can use the same series of events (**GotFocus**/**LostFocus**/**Loaded**) but instead of changing text, let's add a new **TextBlock** into the visual tree of the **PasswordBox** to display our watermark text.

The **PasswordBox** visual control template looks like this (most properties removed for brevity):

```xml
<Grid>
	<Grid.ColumnDefinitions>
		<ColumnDefinition Width="\*" />
		<ColumnDefinition Width="Auto" />
	</Grid.ColumnDefinitions>
	<Border x:Name="BackgroundElement" Grid.ColumnSpan="2"/>
	<Border x:Name="BorderElement" Grid.ColumnSpan="2"/>
	<ScrollViewer x:Name="ContentElement" />
	<Button x:Name="RevealButton" Grid.Column="1" Visibility="Collapsed" />
</Grid>
```

The actual Password text is placed into the **ScrollViewer** named "ContentElement". This is done by the WinRT control itself when it applies the template. What we need is a **TextBlock** to be directly above that **ScrollViewer** displaying our watermark text when the control has no password and does not have focus. As I mentioned before, I'm a big fan of behaviors - so this time I chose to just create a standard attached behavior to accomplish the goal:

```csharp
using JulMar.Windows.Extensions;
using Windows.UI;
using Windows.UI.Xaml;
using Windows.UI.Xaml.Controls;
using Windows.UI.Xaml.Media;

namespace PwBoxWatermark
{
    /// <summary>
    ///     Simple behavior for the PasswordBox to provide a watermark text element.
    /// </summary>
    public static class PasswordBoxBehavior
    {
        private const string WatermarkId = "_pboxWatermark";

        /// <summary>
        ///     Backing storage key for the text property
        /// </summary>
        public static readonly DependencyProperty WatermarkProperty =
            DependencyProperty.RegisterAttached("Watermark", typeof (string), typeof (PasswordBoxBehavior),
                                                new PropertyMetadata("", OnWatermarkChanged));

        /// <summary>
        ///     Gets the watermark text
        /// </summary>
        /// <param name="pbox"></param>
        /// <returns></returns>
        public static string GetWatermark(PasswordBox pbox)
        {
            return (string) pbox.GetValue(WatermarkProperty);
        }

        /// <summary>
        ///     Sets the watermark text
        /// </summary>
        /// <param name="pbox"></param>
        /// <param name="text"></param>
        public static void SetWatermark(PasswordBox pbox, string text)
        {
            pbox.SetValue(WatermarkProperty, text);
        }

        /// <summary>
        ///     Called when the watermark is changed.
        /// </summary>
        /// <param name="dpo"></param>
        /// <param name="e"></param>
        private static void OnWatermarkChanged(DependencyObject dpo, DependencyPropertyChangedEventArgs e)
        {
            var pbox = dpo as PasswordBox;
            if (pbox == null)
                return;

            pbox.PasswordChanged += PboxOnPasswordChanged;
            pbox.GotFocus += PboxOnGotFocus;
            pbox.LostFocus += PboxOnLostFocus;
            pbox.Loaded += PboxOnLoaded;

            string text = (e.NewValue ?? "").ToString();
            if (string.IsNullOrEmpty(text))
            {
                RemoveWatermarkElement(pbox);
            }
            else
            {
                AddWatermarkElement(pbox, text);
            }
        }

        /// <summary>
        ///     Called when the PasswordBox is loaded.  This adds the watermark if one is present.
        /// </summary>
        /// <param name="sender"></param>
        /// <param name="routedEventArgs"></param>
        private static void PboxOnLoaded(object sender, RoutedEventArgs routedEventArgs)
        {
            var pbox = (PasswordBox) sender;
            string text = GetWatermark(pbox);
            if (string.IsNullOrEmpty(text))
            {
                RemoveWatermarkElement(pbox);
            }
            else
            {
                AddWatermarkElement(pbox, text);
            }
        }

        /// <summary>
        ///     Called when the PasswordBox loses focus - this adds the watermark if necessary.
        /// </summary>
        /// <param name="sender"></param>
        /// <param name="routedEventArgs"></param>
        private static void PboxOnLostFocus(object sender, RoutedEventArgs routedEventArgs)
        {
            var pbox = (PasswordBox) sender;
            if (pbox.Password.Length == 0)
            {
                AddWatermarkElement(pbox, GetWatermark(pbox));
            }
        }

        /// <summary>
        ///     Called when the PasswordBox gets focus - this removes any watermark.
        /// </summary>
        /// <param name="sender"></param>
        /// <param name="routedEventArgs"></param>
        private static void PboxOnGotFocus(object sender, RoutedEventArgs routedEventArgs)
        {
            var pbox = (PasswordBox) sender;
            RemoveWatermarkElement(pbox);
        }

        /// <summary>
        ///     This is called when the password is changed in the PasswordBox and removes the watermark.
        /// </summary>
        /// <param name="sender"></param>
        /// <param name="routedEventArgs"></param>
        private static void PboxOnPasswordChanged(object sender, RoutedEventArgs routedEventArgs)
        {
            var pbox = (PasswordBox) sender;
            if (pbox.Password.Length > 0)
            {
                RemoveWatermarkElement(pbox);
            }
        }

        /// <summary>
        ///     Simple method to add a new TextBlock into the visual tree of the
        ///     PasswordBox which will present the watermark.
        /// </summary>
        /// <param name="pbox"></param>
        /// <param name="text"></param>
        private static void AddWatermarkElement(PasswordBox pbox, string text)
        {
            var wmTb = pbox.FindVisualChildByName<TextBlock>(WatermarkId);
            if (wmTb == null)
            {
                var fe = pbox.FindVisualChildByName<ScrollViewer>("ContentElement");
                if (fe != null)
                {
                    var panelOwner = fe.FindVisualParent<Panel>();
                    if (panelOwner != null)
                    {
                        // Add the TextBlock.
                        var tb = new TextBlock
                                     {
                                         Name = WatermarkId,
                                         Text = text,
                                         HorizontalAlignment = HorizontalAlignment.Left,
                                         VerticalAlignment = VerticalAlignment.Center,
                                         Margin = new Thickness(3, 0, 0, 0),
                                         Foreground = new SolidColorBrush(Colors.Gray)
                                     };
                        int index = panelOwner.Children.IndexOf(fe);
                        panelOwner.Children.Insert(index + 1, tb);
                    }
                }
            }
        }

        /// <summary>
        ///     Simple method to remove the TextBlock from the PasswordBox
        ///     visual tree.
        /// </summary>
        /// <param name="pbox"></param>
        private static void RemoveWatermarkElement(PasswordBox pbox)
        {
            var wmTb = pbox.FindVisualChildByName<TextBlock>(WatermarkId);
            if (wmTb != null)
            {
                var panelOwner = wmTb.FindVisualParent<Panel>();
                if (panelOwner != null)
                {
                    panelOwner.Children.Remove(wmTb);
                }
            }
        }
    }
}
```

The only attached property is the **Watermark** property - this activates the behavior and causes the code to attached event handlers to the **GotFocus**, **LostFocus** and **Loaded** events. When the watermark is supposed to be shown (no password, no focus), the code adds a **TextBlock** into the **PasswordBox's** visual tree through the **VisualTreeHelper** class (I'm actually using some extensions from mvvmhelpers.codeplex.com here - added via Nuget **MVVMHelpers.Metro** to do the lookup, but you could replace it with a loop if you want).

To use it, you just apply the **PasswordBoxBehavior.Watermark** property to any **PasswordBox** - it will then display the watermark. Here's an example:

```csharp
<StackPanel HorizontalAlignment="Center" VerticalAlignment="Center">
	<PasswordBox Width="200" local:PasswordBoxBehavior.Watermark="Enter Password..." />
	<Button Content="Login" Margin="20" />
</StackPanel>
```

![](/images/PasswordBoxBehavior.jpg "PasswordBoxBehavior")

[Here's the code if you'd like to use it yourself.](/samples/WatermarkPbox.zip)

Enjoy!
