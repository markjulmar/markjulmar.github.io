---
title: NumericTextBoxBehavior for XAML-based Windows Store Applications
date: 2013-12-11T00:00:00.0000000
layout: post
categories: mvvm
tags: mvvm uwp dotnet
---

One of the first (and arguably most useful) behaviors that I wrote for [MVVMHelpers](https://github.com/markjulmar/mvvmhelpers) was a [NumericTextBoxBehavior](https://github.com/markjulmar/mvvmhelpers/blob/master/Julmar.Wpf.Helpers/Julmar.Wpf.Behaviors/Interactivity/NumericTextBoxBehavior.cs).  This is a WPF behavior that restricts the input for a `TextBox` to be numeric-only.  Recently, someone emailed me asking if it was possible to do the same thing in a Windows Store application.  It absolutely is, but there are some caveats. My original implementation for WPF utilizes _preview events_ - these are events which are sent to the ancestors of the element prior to it receiving a true input event such as `KeyDown`.  The advantage to this approach is we can cancel the input event and make it appear as if the keypress never happened.  For something like a numeric restriction this is perfect - the `TextBox` never even sees non-numeric keyboard events. WinRT, unfortunately, doesn't have preview events - so we have two choices:

1. Use the `TextChanged` event and then reset the text if it's not valid.
1. Extend the control and override the KeyDown logic to strip out numerics.

#2 would provide a better experience, with the downside of no longer being able to use a standard control. #1 allows me to use a behavior-based approach, but you will see the text (briefly) when it is typed. Both will solve the overall issue of only allowing numerics into the field. So, with all that, here's the first pass at a numeric behavior in Windows 8.1 using the Blend Behaviors SDK:

```csharp
using System;
using Windows.UI.Xaml;
using Windows.UI.Xaml.Controls;
using Windows.UI.Xaml.Input;
using Microsoft.Xaml.Interactivity;

namespace App1
{
    ///
    /// Simple NumericTextBox behavior for Windows 8.1
    ///
    public sealed class NumericTextBoxBehavior : DependencyObject, IBehavior
    {
        ///
        /// Track the last valid text value.
        ///
        private string _lastText;

        ///
        /// Backing storage for the AllowDecimal property
        ///
        public static readonly DependencyProperty AllowDecimalProperty = DependencyProperty.Register(
            "AllowDecimal",
            typeof(bool),
            typeof(NumericTextBoxBehavior),
            new PropertyMetadata(false));

        ///
        /// True to allow a decimal point.
        ///
        public bool AllowDecimal
        {
            get
            {
                return (bool)base.GetValue(AllowDecimalProperty);
            }

            set
            {
                base.SetValue(AllowDecimalProperty, value);
            }
        }

        ///
        /// Used to attach this behavior to an element.
        /// Must be a TextBox.
        ///
        ///TextBox to associate this behavior with.
        public void Attach(DependencyObject associatedObject)
        {
            TextBox tb = associatedObject as TextBox;
            if (tb == null)
            {
                throw new ArgumentException("NumericTextBoxBehavior can only be used with a TextBox.");
            }

            AssociatedObject = associatedObject;

            _lastText = tb.Text;
            tb.TextChanged += TbOnTextChanged;
            if (tb.InputScope == null)
            {
                var inputScope = new InputScope();
                inputScope.Names.Add(new InputScopeName(InputScopeNameValue.Number));
                tb.InputScope = inputScope;
            }
        }

        ///
        /// Handles the TextChanged event on the TextBox and watches for
        /// numeric entries.
        ///
        private void TbOnTextChanged(object sender, TextChangedEventArgs e)
        {
            TextBox tb = AssociatedObject as TextBox;
            if (tb != null)
            {
                if (AllowDecimal)
                {
                    double value;
                    if (string.IsNullOrEmpty(tb.Text) || 
                        Double.TryParse(tb.Text, out value))
                    {
                        _lastText = tb.Text;
                        return;
                    }
                }
                else
                {
                    long value;
                    if (string.IsNullOrEmpty(tb.Text) || 
                        long.TryParse(tb.Text, out value))
                    {
                        _lastText = tb.Text;
                        return;
                    }
                }

                tb.Text = _lastText;
                tb.SelectionStart = tb.Text.Length;
            }
        }

        ///
        /// Detaches the behavior from the TextBox.
        ///
        public void Detach()
        {
            TextBox tb = AssociatedObject as TextBox;
            if (tb != null)
            {
                tb.TextChanged -= this.TbOnTextChanged;
            }
        }

        ///
        /// The associated object (TextBox).
        ///
        public DependencyObject AssociatedObject { get; private set; }
    }
}
```

The usage is quite easy - there's only one property you can set (`AllowDecimal`) which controls whether the value can be a double vs. an integer. Here's an example using this behavior:

```xml
<StackPanel HorizontalAlignment="Center" VerticalAlignment="Center">
    <TextBlock Margin="5" FontSize="36"
                Text="Enter an integer value:" />
    <TextBox Width="200" Margin="5">
        <interactivity:Interaction.Behaviors>
            <local:NumericTextBoxBehavior />
        </interactivity:Interaction.Behaviors>
    </TextBox>

    <TextBlock Margin="5" FontSize="36"
                Text="Enter a decimal value:" />
    <TextBox Width="200" Margin="5">
        <interactivity:Interaction.Behaviors>
            <local:NumericTextBoxBehavior AllowDecimal="true" />
        </interactivity:Interaction.Behaviors>
    </TextBox>
</StackPanel>
```

If you place this code into a blank Windows Store app application, it will restrict the text box fields to an integer and decimal value - however typing a non-numeric into the field will briefly show the character before it is caught and removed by the behavior.

[Here's the sample if you want to try it yourself](/samples/NumericTextBox.Win81.zip)

While writing this, I realized you could abstract this approach a little further and create a more diverse set of `TextBox` behaviors - to support other transformations at the view level.  More on that in the next post!
