---
title: Playing with WPF Behaviors - a WatermarkText behavior
date: 2009-07-24T00:00:00.0000000
layout: post
categories: mvvm
tags: mvvm wpf dotnet
---

One of the coolest new features of Blend 3 is the inclusion of behaviors. This new feature formalizes the "attached behavior" model that has become so prevalent in WPF (and Silverlight) development.  I won't go into details on the architecture - instead I'll refer you to a nice reference:

[http://blogs.msdn.com/expression/archive/2009/05/19/link-round-up-behaviors-related-posts.aspx](http://blogs.msdn.com/expression/archive/2009/05/19/link-round-up-behaviors-related-posts.aspx)

To play with this new support, I built a WatermarkTextBehavior which places a watermark into a TextBox when it has no entered data.  I've included this new behavior into the current build of my MVVM toolkit which I'll release soon, but for now, let's look at the behavior:

First, we derive from `System.Windows.Interactivity.Behavior<T>` - the placeholder parameter is the Visual type you want the behavior to act on.  This can be **UIElement** for anything WPF, or more restrictive if necessary based on the events you intend to hook up.  For our purposes here, we will set the restricted type to **TextBox**.

```csharp
public class WatermarkTextBehavior : Behavior<TextBox>
```

Next, you override the **OnAttached()** and **OnDetaching()** methods to hook up your event behaviors you desire.  Call the base implementation first, and then the **AssociatedObject** property will be the element you've been attached to (the **TextBox** in this case).  In our case we want to hook the **GotFocus** and **LostFocus** events - this is where we trigger our behavior.

```csharp
protected override void OnAttached()
{
    base.OnAttached();
    AssociatedObject.GotFocus += OnGotFocus;
    AssociatedObject.LostFocus += OnLostFocus;
    ...  
}

protected override void OnDetaching()
{  
    base.OnDetaching();  
    AssociatedObject.GotFocus -= OnGotFocus;  
    AssociatedObject.LostFocus -= OnLostFocus;  
}
```

Finally, we can provide any properties necessary to drive our behavior.  These should be done in the form of Dependency Properties so they are bindable and interact nicely with WPF.  The base `Behavior<T>` derives from **Freezable** and inherits the DataContext automatically to enable this support.  In our implementation we will have a Text property to indicate the watermark, and an attached property which we will place onto the TextBox so it can be styled when the watermark is being used.

```csharp
public static readonly DependencyProperty TextProperty = DependencyProperty.Register("Text", typeof (string), 
        typeof (WatermarkTextBehavior), new FrameworkPropertyMetadata(string.Empty));
```

Next we can hook it up in Blend through the Asset panel - all known assets are shown here (either registered, in the Blend directory, or project references).  Drag our **WatermarkTextBehavior** onto any **TextBox** and set the **Text** property and it will generate the following XAML:

```xml
<TextBox>
  <TextBox.Style>
    <Style TargetType="TextBox" BasedOn="{StaticResource {x:Type TextBox}}">
      <Style.Triggers>
        <Trigger Property="julmar:WatermarkTextBehavior.IsWatermarked" Value="True">
          <Setter Property="Foreground" Value="Gray"/>
          <Setter Property="FontStyle" Value="Italic"/>
        </Trigger>
      </Style.Triggers>
    </Style>
  </TextBox.Style>
  <i:Interaction.Behaviors>
    <julmar:WatermarkTextBehavior Text="Enter a name here"/>
  </i:Interaction.Behaviors>
</TextBox>
```

Here is the complete source code to the behavior:

```csharp
using System;
using System.Windows;
using System.Windows.Controls;
using System.Windows.Interactivity;

namespace JulMar.Windows.Interactivity
{
    /// <summary>
    /// This behavior associates a watermark onto a TextBox indicating what the user should
    /// provide as input.
    /// </summary>
    public class WatermarkTextBehavior : Behavior<TextBox>
    {
        /// <summary>
        /// The watermark text
        /// </summary>
        public static readonly DependencyProperty TextProperty =
            DependencyProperty.Register("Text", typeof (string), typeof (WatermarkTextBehavior),
                                        new FrameworkPropertyMetadata(string.Empty));

        static readonly DependencyPropertyKey IsWatermarkedPropertyKey =
            DependencyProperty.RegisterAttachedReadOnly("IsWatermarked", typeof(bool), typeof(WatermarkTextBehavior), 
                                        new FrameworkPropertyMetadata(false));

        /// <summary>
        /// This readonly property is applied to the TextBox and indicates whether the watermark
        /// is currently being displayed.  It allows a style to change the visual appearanve of the
        /// TextBox.
        /// </summary>
        public static readonly DependencyProperty IsWatermarkedProperty = IsWatermarkedPropertyKey.DependencyProperty;

        /// <summary>
        /// Retrieves the current watermarked state of the TextBox.
        /// </summary>
        /// <param name="tb"></param>
        /// <returns></returns>
        public static bool GetIsWatermarked(TextBox tb)
        {
            return (bool) tb.GetValue(IsWatermarkedProperty);
        }

        /// <summary>
        /// Retrieves the current watermarked state of the TextBox.
        /// </summary>
        public bool IsWatermarked
        {
            get { return GetIsWatermarked(AssociatedObject);  }    
            private set { AssociatedObject.SetValue(IsWatermarkedPropertyKey, value);}
        }

        /// <summary>
        /// The watermark text
        /// </summary>
        public string Text
        {
            get { return (string) base.GetValue(TextProperty); }
            set { base.SetValue(TextProperty, value); }
        }

        /// <summary>
        /// Called after the behavior is attached to an AssociatedObject.
        /// </summary>
        /// <remarks>
        /// Override this to hook up functionality to the AssociatedObject.
        /// </remarks>
        protected override void OnAttached()
        {
            base.OnAttached();
            AssociatedObject.GotFocus += OnGotFocus;
            AssociatedObject.LostFocus += OnLostFocus;

            OnLostFocus(null, null);
        }

        /// <summary>
        /// Called when the behavior is being detached from its AssociatedObject, but before it has actually occurred.
        /// </summary>
        /// <remarks>
        /// Override this to unhook functionality from the AssociatedObject.
        /// </remarks>
        protected override void OnDetaching()
        {
            base.OnDetaching();
            AssociatedObject.GotFocus -= OnGotFocus;
            AssociatedObject.LostFocus -= OnLostFocus;
        }

        /// <summary>
        /// This method is called when the textbox gains focus.  It removes the watermark.
        /// </summary>
        /// <param name="sender"></param>
        /// <param name="e"></param>
        private void OnGotFocus(object sender, RoutedEventArgs e)
        {
            if (string.Compare(AssociatedObject.Text, this.Text, StringComparison.OrdinalIgnoreCase) == 0)
            {
                AssociatedObject.Text = string.Empty;
                IsWatermarked = false;
            }
        }

        /// <summary>
        /// This method is called when focus is lost from the TextBox.  It puts the watermark
        /// into place if no text is in the textbox.
        /// </summary>
        /// <param name="sender"></param>
        /// <param name="e"></param>
        private void OnLostFocus(object sender, RoutedEventArgs e)
        {
            if (string.IsNullOrEmpty(AssociatedObject.Text))
            {
                AssociatedObject.Text = this.Text;
                IsWatermarked = true;
            }
        }
    }
}
```
