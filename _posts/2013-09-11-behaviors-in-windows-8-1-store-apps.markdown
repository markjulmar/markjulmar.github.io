---
title: Behaviors in Windows 8.1 Store Apps
date: 2013-09-11T00:00:00.0000000
layout: post
categories: mvvm
tags: mvvm dotnet
---

Visual Studio 2013 RC was released to MSDN this week and it includes some very cool new features which have been detailed on various blogs around the Internet.  One feature I've not seen much coverage of, but am _very_ excited about is support for Blend behaviors.  If you create a new Windows Store application using the 8.1 profile you will find Blend behaviors in the Assets tab:

[](/images/Blend-Behaviors.jpg "Blend-Behaviors")

The list of behaviors is small - compared to WPF and Silverlight, but all the important ones are there, along with a new one - `IncrementalUpdateBehavior` which I'll get to in a later post.

What I wanted to talk about here was the _architecture_ of these behaviors.  First, they are in an SDK assembly - so not in the core .NET profile.  In order to use them, you either need to pull up Blend and drag one onto your design surface (which adds the appropriate references), or add the Behaviors SDK reference in Visual Studio 2013:

[](/images/AddReference-Behaviors.jpg "AddReference-Behaviors")

This will add two assemblies to your project which come from the Windows SDK folder:

![](/images/Folder-Behaviors.jpg "Folder-Behaviors")

`Microsoft.Xaml.Interactivity` is where the core support is - any custom behaviors will need access to the types in this assembly.  `Microsoft.Xaml.Interactions` is where the system-supplied behaviors are located.

![](/images/Class-List-Behaviors.jpg "Class-List-Behaviors")

Examining the core data structures, you will see several differences from the WPF/SL behavior world - in particular there is no set of base classes (`Behavior<T>`, `Trigger<T>`, `Action<T>`) to derive from - instead we have some interfaces: `IAction` and `IBehavior`.  This is presumably because WinRT doesn't support inheritance across the ABI and they want to keep the same basic style for native behaviors in the C++ world.

Notice as well, there is no _trigger_ in this implementation - instead, everything is a _behavior_ and some behaviors contain a collection of _actions_ to invoke.  So they've simplified the model some.  So, let's see how we'd implement a custom behavior in this new world.

As our example, we'll re implement the `DragPositionBehavior` which is missing in this framework. First, we create a type that derives from `DependencyObject` and implement `IBehavior`.  `DependencyObject` will give us access to dependency properties which are bindable - and the infrastructure will ensure that the DataContext is carried through for us.  In my experimentation, I found that deriving from higher level constructs such as `FrameworkElement` stopped some things from working so it appears that `DependencyObject` is the best base class for this at the moment.

```csharp
public class DragPositionBehavior : DependencyObject, IBehavior
{
}
```

Next, add any properties which will control this behavior - we will get to triggers and actions in a moment, but for this example we only need a flag as to whether this is enabled or not:

```csharp
public class DragPositionBehavior : DependencyObject, IBehavior
{
    /// <summary>
    /// Backing storage for the IsEnabled property.
    /// </summary>
    public static DependencyProperty IsEnabledProperty = DependencyProperty.Register("IsEnabled", typeof(bool), typeof(DragPositionBehavior), new PropertyMetadata(false));

    /// <summary>
    /// Property to enable/disable the drag capability
    /// </summary>
    public bool IsEnabled
    {
        get { return (bool) base.GetValue(IsEnabledProperty); }
        set { base.SetValue(IsEnabledProperty, value); }
    }
}
```

Next, we need to implement the `IBehavior` interface. It looks like this:

```csharp
public interface IBehavior
{
    DependencyObject AssociatedObject { get; }
    void Attach(DependencyObject associatedObject);
    void Detach();
}
```

This should look a bit familiar if you've built behaviors for other XAML based platforms - it's the basic interface used by the Behavior class. In this case, the system will call `Attach()` when our behavior is associated with an element, and `Detach()` when it is removed. Your code is responsible for setting the AssociatedObject property - notice it only has a getter in the interface. Here's a bare-bones implementation that will hook into the PointerPressed event when it is attached and remove our handler when it is released, along the way we'll have a little logic to verify we aren't added to more than one element, not supported in design-mode (important!) and we are always detached before being re-attached:

```csharp
/// <summary>
/// Get the associated target object
/// </summary>
public DependencyObject AssociatedObject { get; private set; }

/// <summary>
/// Attaches to the specified object.
/// </summary>
public void Attach(DependencyObject associatedObject)
{
    if ((associatedObject != AssociatedObject)
        && !DesignMode.DesignModeEnabled)
    {
        if (AssociatedObject != null)
            throw new InvalidOperationException("Cannot attach behavior multiple times.");

        AssociatedObject = associatedObject;
        FrameworkElement fe = AssociatedObject as FrameworkElement;
        if (fe != null)
        {
            fe.AddHandler(UIElement.PointerPressedEvent, new PointerEventHandler(OnPointerPressed), true);
        }
    }
}

/// <summary>
/// Detaches this instance from its associated object.
/// </summary>
public void Detach()
{
    FrameworkElement fe = AssociatedObject as FrameworkElement;
    if (fe != null)
    {
        fe.RemoveHandler(UIElement.PointerPressedEvent, new PointerEventHandler(OnPointerPressed));
    }

    AssociatedObject = null;
}
```

Next, let's add our pointer logic to watch for a touch-drag or left-button-down drag:

```csharp
/// <summary>
/// Handles the pointer down event
/// </summary>
private void OnPointerPressed(object sender, PointerRoutedEventArgs e)
{
    // Ignore if the property is turned off.
    if (!IsEnabled) return;

    // Begin the drag operation
    FrameworkElement uie = (FrameworkElement) sender;
    if (e.Pointer.PointerDeviceType == PointerDeviceType.Mouse) {
        // If it's not the left button, then exit.
        if (!e.GetCurrentPoint(uie).Properties.IsLeftButtonPressed)
            return;
    }

    var currentDragInfo = new ElementMouseDrag();
    currentDragInfo.OnPointerDown(uie, e);
}
```

That's the basic implementation - Attach hooks any events, Detach unhooks events and then you provide event handlers to drive your behavior. Here's the full sample:

```csharp
using System;
using Windows.ApplicationModel;
using Windows.Devices.Input;
using Windows.UI.Input;
using Windows.UI.Xaml;
using Windows.UI.Xaml.Controls;
using Windows.UI.Xaml.Input;
using Windows.UI.Xaml.Media;
using Microsoft.Xaml.Interactivity;

namespace JulMar.Windows.Interactivity
{
    /// <summary>
    /// This Blend behavior provides positional translation for elements through a
    /// RenderTransform using Drag/Drop semantics.
    /// </summary>
    public class DragPositionBehavior : DependencyObject, IBehavior
    {
        /// <summary>
        /// Backing storage for the IsEnabled property.
        /// </summary>
        public static DependencyProperty IsEnabledProperty = DependencyProperty.Register("IsEnabled", typeof (bool), typeof (DragPositionBehavior), new PropertyMetadata(false));

        /// <summary>
        /// Property to enable/disable the drag capability
        /// </summary>
        public bool IsEnabled
        {
            get { return (bool) base.GetValue(IsEnabledProperty); }
            set { base.SetValue(IsEnabledProperty, value); }
        }

        /// <summary>
        /// Get the associated target object
        /// </summary>
        public DependencyObject AssociatedObject { get; private set; }

        /// <summary>
        /// Attaches to the specified object.
        /// </summary>
        public void Attach(DependencyObject associatedObject)
        {
            if ((associatedObject != AssociatedObject)
                && !DesignMode.DesignModeEnabled)
            {
                if (AssociatedObject != null)
                    throw new InvalidOperationException("Cannot attach behavior multiple times.");

                AssociatedObject = associatedObject;
                FrameworkElement fe = AssociatedObject as FrameworkElement;
                if (fe != null)
                {
                    fe.AddHandler(UIElement.PointerPressedEvent, new PointerEventHandler(OnPointerPressed), true);
                }
            }
        }

        /// <summary>
        /// Detaches this instance from its associated object.
        /// </summary>
        public void Detach()
        {
            FrameworkElement fe = AssociatedObject as FrameworkElement;
            if (fe != null)
            {
                fe.RemoveHandler(UIElement.PointerPressedEvent, new PointerEventHandler(OnPointerPressed));
            }
            AssociatedObject = null;
        }

        /// <summary>
        /// Handles the pointer down event
        /// </summary>
        private void OnPointerPressed(object sender, PointerRoutedEventArgs e)
        {
            // Ignore if the property is turned off.
            if (!IsEnabled) return;

            // Begin the drag operation
            FrameworkElement uie = (FrameworkElement) sender;
            if (e.Pointer.PointerDeviceType == PointerDeviceType.Mouse) {
                // If it's not the left button, then exit.
                if (!e.GetCurrentPoint(uie).Properties.IsLeftButtonPressed)
                    return;
            }

            var currentDragInfo = new ElementMouseDrag();
            currentDragInfo.OnPointerDown(uie, e);
        }

        /// <summary>
        /// This class encapsulates the drag data + logic for a given element.
        /// It saves memory space as it is only allocated while the object is
        /// being dragged around.
        /// </summary>
        internal class ElementMouseDrag
        {
            private PointerPoint _startPos;
            private bool _adjustCanvasCoordinates, _movedItem;
            private TranslateTransform _translatePos;

            public void OnPointerDown(FrameworkElement uie, PointerRoutedEventArgs e)
            {
                FrameworkElement parent = VisualTreeHelper.GetParent(uie) as FrameworkElement;
                _adjustCanvasCoordinates = (parent != null && parent.GetType() == typeof(Canvas));
                _startPos = e.GetCurrentPoint(null);

                uie.PointerReleased += OnPointerReleased;
                uie.PointerMoved += OnPointerMoved;
                uie.PointerCaptureLost += OnLostCapture;
                uie.CapturePointer(e.Pointer);
            }

            private void OnPointerReleased(object sender, PointerRoutedEventArgs e)
            {
                FrameworkElement uie = (FrameworkElement)sender;
                uie.ReleasePointerCapture(e.Pointer);
                if (_movedItem)
                    e.Handled = true;
            }

            private void OnLostCapture(object sender, PointerRoutedEventArgs e)
            {
                FrameworkElement uie = (FrameworkElement)sender;
                uie.PointerReleased -= OnPointerReleased;
                uie.PointerMoved -= OnPointerMoved;
            }

            public void OnPointerMoved(object sender, PointerRoutedEventArgs e)
            {
                FrameworkElement uie = (FrameworkElement)sender;

                PointerPoint currentPosition = e.GetCurrentPoint(null);

                if (!_movedItem)
                {
                    if (Math.Abs(currentPosition.Position.X - _startPos.Position.X) <= 5 &&
                        Math.Abs(currentPosition.Position.Y - _startPos.Position.Y) <= 5)
                        return;
                }

                _movedItem = true;
                e.Handled = true;

                if (_adjustCanvasCoordinates)
                {
                    double adjustX = currentPosition.Position.X - _startPos.Position.X;
                    double adjustY = currentPosition.Position.Y - _startPos.Position.Y;

                    Canvas.SetLeft(uie, Canvas.GetLeft(uie) + adjustX);
                    Canvas.SetTop(uie, Canvas.GetTop(uie) + adjustY);

                    _startPos = currentPosition;
                }
                else // Not in a Canvas - use a RenderTransform instead.
                {
                    if (_translatePos != null)
                    {
                        _translatePos.X = currentPosition.Position.X - _startPos.Position.X;
                        _translatePos.Y = currentPosition.Position.Y - _startPos.Position.Y;
                    }
                    else
                    {
                        _translatePos = new TranslateTransform { X = currentPosition.Position.X - _startPos.Position.X, Y = currentPosition.Position.Y - _startPos.Position.Y };

                        // Replace existing transform if it exists.
                        TransformGroup transformGroup = new TransformGroup();
                        if (uie.RenderTransform != null)
                            transformGroup.Children.Add(uie.RenderTransform);
                        transformGroup.Children.Add(_translatePos);
                        uie.RenderTransform = transformGroup;
                    }
                }
            }
        }
    }
}
```

Ok, so that's basic behaviors - what about triggers and actions? Well, as I mentioned, triggers are missing from this framework and have been replaced by behaviors with an Actions collection. There are two supplied trigger behaviors:

- `EventTriggerBehavior` runs a set of actions when an event occurs - typically from the UI.
- `DataTriggerBehavior` runs a set of actions when a binding value matches a specific condition (==, !=, >, etc.)

As with normal behaviors, we can create our own trigger behaviors. As an example, let's implement a `TimerTrigger` (also missing from the basic set of behaviors). The big difference between this and the prior example is we now have a `DependencyProperty` which returns an `ActionCollection` to assign actions to, and at some point we call the built-in `Interaction.ExecuteActions` method to run the actions.

Here's the code:

```csharp
using System;
using Windows.ApplicationModel;
using Windows.UI.Xaml;
using Windows.UI.Xaml.Markup;
using Microsoft.Xaml.Interactivity;

namespace JulMar.Windows.Interactivity
{
    /// <summary>
    /// Trigger which runs off a timer
    /// </summary>
    [ContentProperty(Name = "Actions")]
    public sealed class TimerTriggerBehavior : DependencyObject, IBehavior
    {
        private int _tickCount;
        private DispatcherTimer _timer;

        /// <summary>
        /// Backing storage for Actions collection
        /// </summary>
        public static readonly DependencyProperty ActionsProperty = DependencyProperty.Register("Actions", typeof(ActionCollection), typeof(TimerTriggerBehavior), new PropertyMetadata(null));

        /// <summary>
        /// Actions collection
        /// </summary>
        public ActionCollection Actions
        {
            get
            {
                ActionCollection actions = (ActionCollection)base.GetValue(ActionsProperty);
                if (actions == null)
                {
                    actions = new ActionCollection();
                    base.SetValue(ActionsProperty, actions);
                }
                return actions;
            }
        }

        /// <summary>
        /// Backing storage for the counter
        /// </summary>
        public static readonly DependencyProperty MillisecondsPerTickProperty = DependencyProperty.Register("MillisecondsPerTick", typeof(double), typeof(TimerTrigger), new PropertyMetadata(1000.0));

        /// <summary>
        /// Milliseconds
        /// </summary>
        public double MillisecondsPerTick
        {
            get
            {
                return (double)base.GetValue(MillisecondsPerTickProperty);
            }
            set
            {
                base.SetValue(MillisecondsPerTickProperty, value);
            }
        }

        /// <summary>
        /// Backing storage for the total ticks counter
        /// </summary>
        public static readonly DependencyProperty TotalTicksProperty = DependencyProperty.Register("TotalTicks", typeof(int), typeof(TimerTrigger), new PropertyMetadata(-1));

        /// <summary>
        /// Total ticks elapsed
        /// </summary>
        public int TotalTicks
        {
            get
            {
                return (int)base.GetValue(TotalTicksProperty);
            }
            set
            {
                base.SetValue(TotalTicksProperty, value);
            }
        }

        /// <summary>
        /// Called when the timer elapses
        /// </summary>
        /// <param name="sender"></param>
        /// <param name="e"></param>
        private void OnTimerTick(object sender, object e)
        {
            if (this.TotalTicks > 0 && ++this._tickCount >= this.TotalTicks)
            {
                this.StopTimer();
            }

            // Raise the actions
            Interaction.ExecuteActions(AssociatedObject, this.Actions, null);
        }

        /// <summary>
        /// Called to start the timer
        /// </summary>
        internal void StartTimer()
        {
            this._timer = new DispatcherTimer {Interval = (TimeSpan.FromMilliseconds(this.MillisecondsPerTick))};
            this._timer.Tick += this.OnTimerTick;
            this._timer.Start();
        }

         /// <summary>
        /// Called to stop the timer
        /// </summary>
        internal void StopTimer()
        {
            if (this._timer != null)
            {
                this._timer.Stop();
                this._timer = null;
            }
        }

        /// <summary>
        /// Attaches to the specified object.
        /// </summary>
        /// <param name="associatedObject">The <see cref="T:Windows.UI.Xaml.DependencyObject"/> to which the <seealso cref="T:Microsoft.Xaml.Interactivity.IBehavior"/> will be attached.</param>
        public void Attach(DependencyObject associatedObject)
        {
            if ((associatedObject != AssociatedObject) && !DesignMode.DesignModeEnabled)
            {
                if (AssociatedObject != null)
                    throw new InvalidOperationException("Cannot attach behavior multiple times.");
                AssociatedObject = associatedObject;
                StartTimer();
            }
        }

        /// <summary>
        /// Detaches this instance from its associated object.
        /// </summary>
        public void Detach()
        {
            this.StopTimer();
            AssociatedObject = null;
        }

        /// <summary>
        /// Gets the <see cref="T:Windows.UI.Xaml.DependencyObject"/> to which the <seealso cref="T:Microsoft.Xaml.Interactivity.IBehavior"/> is attached.
        /// </summary>
        public DependencyObject AssociatedObject { get; private set; }
    }
}
```

A couple of notes.

1. The `ContentPropertyAttribute` applied to the class allows us to add `Action` elements directly as children to the behavior. This is important, both for Blendability but also for the action collection management.
1. The `Actions` property creates a new `ActionCollection` if one doesn't exist in the getter. This is a requirement - otherwise WinRT throws a weird null collection exception.
1. When the timer expires, we call the static method `Interaction.ExecuteActions` which takes the object raising the notification, the ActionCollection and a parameter which will be passed to each action. This method is built into the framework and will do the work of running all the IAction types.

To use the behavior, you can pull up your code in Blend, build the app (so Blend sees the behaviors - note you will need a reference to the Behaviors SDK or it won't see them until you have it).  Drag the `TimerTriggerBehavior` onto an element and then drag actions onto the behavior - this applies the actions. The default trigger (when you drag/drop and action by itself onto an element) is an `EventTriggerBehavior`. You can also just type the code into Visual Studio 2013 (make sure to define the appropriate namespaces).

```xml
<!--Play sound after 1 second-->
<interactivity1:TimerTriggerBehavior MillisecondsPerTick="1000" TotalTicks="2">
    <media:PlaySoundAction Source="tada.wav" Volume="1" />
</interactivity1:TimerTriggerBehavior>
```

Lastly, we have actions. These are bits of code to perform when a trigger behavior runs. This is actually the main thing people typically build UI behaviors on top of.

There are several system-supplied actions including

- `CallMethodAction` - invokes a public method, typically on a viewmodel
- `ChangePropertyAction` - changes a property value
- `ControlStoryboardAction` - play, pause, stop a storyboard animation
- `GoToStateAction` - changes a visual state using the VisualStateManager
- `InvokeCommandAction` - invokes an `ICommand` implementation, typically on a viewmodel
- `NavigateToPageAction` - does a frame Navigate to a specific page
- `PlaySoundAction` - plays a media file.

You can easily create your own - in this case it requires implementing `IAction` which looks like:

```csharp
public interface IAction
{
    object Execute(object sender, object parameter);
}
```

Very simple. As an example, let's implement a `SetFocusAction` to push focus to a UI element, it doesn't have an Attach/Detach - it simply executes. However, it also should derive from `DependencyObject` and expose any controlling properties as `DependencyProperty` types for binding purposes. In our example here, we need a UI target to shift focus to when the action runs so we will define a `Target` property and then when we are invoked, attempt to move focus programmatically to that target.

```csharp
using Microsoft.Xaml.Interactivity;
using Windows.UI.Xaml;
using Windows.UI.Xaml.Controls;

namespace JulMar.Windows.Interactivity
{
    /// <summary>
    /// Shifts focus to some control
    /// </summary>
    public sealed class SetFocusAction : DependencyObject, IAction
    {
        /// <summary>
        /// Backing storage for Target property
        /// </summary>
        public static readonly DependencyProperty TargetProperty = DependencyProperty.Register("Target", typeof (Control), typeof (SetFocusAction), new PropertyMetadata(null));

        /// <summary>
        /// The focus target.
        /// </summary>
        public Control Target
        {
            get { return (Control)base.GetValue(TargetProperty); }
            set { base.SetValue(TargetProperty, value); }
        }

        /// <summary>
        /// Executes the action.
        /// </summary>
        /// <param name="sender">The <see cref="T:System.Object"/> that is passed to the action by the behavior. Generally this is <seealso cref="P:Microsoft.Xaml.Interactivity.IBehavior.AssociatedObject"/> or a target object.</param><param name="parameter">The value of this parameter is determined by the caller.</param>
        /// <remarks>
        /// An example of parameter usage is EventTriggerBehavior, which passes the EventArgs as a parameter to its actions.
        /// </remarks>
        /// <returns>
        /// Returns the result of the action.
        /// </returns>
        public object Execute(object sender, object parameter)
        {
            if (Target != null)
            {
                return Target.Focus(FocusState.Programmatic);
            }
            return false;
        }
    }
}
```

If you are interested in more Behaviors, TriggerBehaviors and Actions, check out [MVVMHelpers](https://github.com/markjulmar/mvvmhelpers/) - it has several additional built-in behaviors for 8.1, along with the same behaviors (and all the plumbing) for Windows 8.  You can also just add it directly to your project with NuGet by searching for "MVVMHelpers.Metro".
