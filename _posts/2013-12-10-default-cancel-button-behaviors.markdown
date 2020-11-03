---
title: Default and Cancel button behaviors
date: 2013-12-10T00:00:00.0000000
layout: post
categories: mvvm
tags: mvvm dotnet
---

One of the things I miss from WPF moving to Windows Store Apps is the ability to define a "Cancel" and "Default" button.  These are buttons which are automatically invoked when you press ENTER or ESC.  A prompt from a fellow developer got me to thinking about how we could accomplish this with an attached behavior until Microsoft decides to add the support into the framework.  Here's the usage I wanted:

```xml
<Button Content="Cancel" Margin="5" Click="OnCancel" behaviors:ButtonBehavior.IsCancel="true" />
<Button Content="Default" Margin="5" Click="OnClick" behaviors:ButtonBehavior.IsDefault="true" />
```

A few rules:

- It should not invoke the default or cancel buttons if they are disabled.
- It should not invoke the default button if focus is in a multi-line TextBox.
- It should detect the keypresses globally on the page, but unsubscribe if the page is navigated away.

Here's the behavior I came up with:

```csharp
using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;
using System.Threading.Tasks;
using Windows.UI.Xaml;
using Windows.UI.Xaml.Controls;
using Windows.UI.Xaml.Controls.Primitives;
using JulMar.Windows.Extensions;
using Windows.UI.Xaml.Automation.Peers;
using Windows.UI.Xaml.Automation.Provider;
using Windows.UI.Core;
using Windows.System;
using Windows.UI.Xaml.Input;

namespace App1.Behaviors
{
    /// <summary>
    /// This class adds the IsDefault and IsCancel properties to buttons.
    /// </summary>
    public static class ButtonBehavior
    {
        private static readonly DependencyProperty DefaultButtonProperty = DependencyProperty.RegisterAttached("__DefaultButtonP__", typeof(Button), typeof(ButtonBehavior), new PropertyMetadata(null));

        public static readonly DependencyProperty IsDefaultProperty = DependencyProperty.RegisterAttached("IsDefault", typeof(bool), typeof(ButtonBehavior), new PropertyMetadata(false, OnIsDefaultChanged));

        public static bool GetIsDefault(Button button) { return (bool) button.GetValue(IsDefaultProperty); }
        public static void SetIsDefault(Button button, bool value) { button.SetValue(IsDefaultProperty, value); }

        private static void OnIsDefaultChanged(DependencyObject sender, DependencyPropertyChangedEventArgs e)
        {
            Button button = sender as Button;
            if (button == null)
                return;

            // Find the page this button is on.
            Page owner = button.FindVisualParent<Page>();
            if (owner == null)
            {
                RoutedEventHandler eh = null;
                eh = (_s, _e) => {
                    button.Loaded -= eh;
                    InitializeButton(button, true, true);
                };
                button.Loaded += eh;
            }
            else InitializeButton(button, (bool)e.NewValue, true);
        }

        private static readonly DependencyProperty CancelButtonProperty = DependencyProperty.RegisterAttached("__CancelButtonP__", typeof(Button), typeof(ButtonBehavior), new PropertyMetadata(null));

        public static readonly DependencyProperty IsCancelProperty = DependencyProperty.RegisterAttached("IsCancel", typeof(bool), typeof(ButtonBehavior), new PropertyMetadata(false, OnIsCancelChanged));

        public static bool GetIsCancel(Button button) { return (bool)button.GetValue(IsCancelProperty); }
        public static void SetIsCancel(Button button, bool value) { button.SetValue(IsCancelProperty, value); }

        private static void OnIsCancelChanged(DependencyObject sender, DependencyPropertyChangedEventArgs e)
        {
            Button button = sender as Button;
            if (button == null)
                return;

            // Find the page this button is on.
            Page owner = button.FindVisualParent<Page>();
            if (owner == null)
            {
                RoutedEventHandler eh = null;
                eh = (_s, _e) => {
                    button.Loaded -= eh;
                    InitializeButton(button, true, false);
                };
                button.Loaded += eh;
            }
            else InitializeButton(button, (bool)e.NewValue, false);
        }

        private static void InitializeButton(Button button, bool attach, bool isDefault)
        {
            Page owner = button.FindVisualParent<Page>();
            if (owner == null)
                return;

            owner.Unloaded += (_s, _e) => {
                Window.Current.CoreWindow.Dispatcher.AcceleratorKeyActivated -= CoreDispatcher_AcceleratorKeyActivated;
            };

            owner.ClearValue(DefaultButtonProperty);
            Window.Current.CoreWindow.Dispatcher.AcceleratorKeyActivated -= CoreDispatcher_AcceleratorKeyActivated;
            if (isDefault)
                button.ClearValue(Button.BorderThicknessProperty);

            if (attach == true)
            {
                Window.Current.CoreWindow.Dispatcher.AcceleratorKeyActivated += CoreDispatcher_AcceleratorKeyActivated;
                if (isDefault) {
                    owner.SetValue(DefaultButtonProperty, button);
                    button.BorderThickness = new Thickness(2);
                } else {
                    owner.SetValue(CancelButtonProperty, button);
                }
            }
        }

        private static void CoreDispatcher_AcceleratorKeyActivated(Windows.UI.Core.CoreDispatcher sender, Windows.UI.Core.AcceleratorKeyEventArgs e)
        {
            var downState = CoreVirtualKeyStates.Down;
            var coreWindow = Window.Current.CoreWindow;
            bool menuKey = (coreWindow.GetKeyState(VirtualKey.Menu) & downState) == downState;
            bool controlKey = (coreWindow.GetKeyState(VirtualKey.Control) & downState) == downState;
            bool shiftKey = (coreWindow.GetKeyState(VirtualKey.Shift) & downState) == downState;
            bool noModifiers = !menuKey && !controlKey && !shiftKey;

            if (noModifiers && e.EventType == Windows.UI.Core.CoreAcceleratorKeyEventType.KeyDown 
                && (e.VirtualKey == Windows.System.VirtualKey.Enter || e.VirtualKey == Windows.System.VirtualKey.Escape))
            {
                Frame frame = Window.Current.Content as Frame;
                if (frame == null) return;

                Page currentPage = frame.Content as Page;
                if (currentPage == null)
                    return;

                if (e.VirtualKey == Windows.System.VirtualKey.Enter)
                {
                    // Quick check to avoid TextBox with ENTER support
                    var tb = FocusManager.GetFocusedElement() as TextBox;
                    if (tb != null && tb.AcceptsReturn) return;

                    Button defaultButton = currentPage.GetValue(DefaultButtonProperty) as Button;
                    if (defaultButton != null && defaultButton.IsEnabled)
                    {
                        ButtonAutomationPeer peer = new ButtonAutomationPeer(defaultButton);
                        IInvokeProvider invokeProv = peer.GetPattern(PatternInterface.Invoke) as IInvokeProvider;
                        if (invokeProv != null)
                            invokeProv.Invoke();
                    }
                }
                else
                {
                    Button cancelButton = currentPage.GetValue(CancelButtonProperty) as Button;
                    if (cancelButton != null && cancelButton.IsEnabled)
                    {
                        ButtonAutomationPeer peer = new ButtonAutomationPeer(cancelButton);
                        IInvokeProvider invokeProv = peer.GetPattern(PatternInterface.Invoke) as IInvokeProvider;
                        if (invokeProv != null)
                            invokeProv.Invoke();
                    }
                }
            }
        }
    }
}
```

[Here's the sample code if you want to try it yourself](/samples/ButtonBehaviors.WinRT.zip)

Tell me what you think!
