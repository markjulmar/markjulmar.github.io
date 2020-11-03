---
title: ViewModel triggers
date: 2013-11-19T00:00:00.0000000
layout: post
categories: dotnet
tags: dotnet
---

A question I often get is how to drive behavior through MVVM where the activity is actually initiated on the ViewModel side. It's easy to see the other direction - where the UI raises a command or property change to cause something to happen.

I often use one of two approaches when initiating state changes in the UI from the ViewModel. The first is to use property changes - since Data Binding is all about property change notification, you can easily make property changes and cause things to happen in the UI - where elements enable / disable, appear / disappear, content gets changed or added / removed, etc. You can even drive full page changes using Data Templates and a `ContentControl` data bound to a view model itself - changing the view model changes the entire UI presented. This can be a handy technique for "wizard" style sheets where you progress through stages in your UI.

Another approach I've used is to use the Visual State Manager, wired up to a [ViewModelTrigger](https://github.com/markjulmar/mvvmhelpers/blob/master/Julmar.Wpf.Helpers/Julmar.Wpf.Behaviors/Triggers/ViewModelTrigger.cs). The idea is to have an event on the ViewModel, which when raised, causes something to happen in the UI. I built a custom Blend trigger (included in [MVVMHelpers](https://github.com/markjulmar/mvvmhelpers)) for this which is used like this:

```xml
<i:Interaction.Triggers>
    <mvvm:ViewModelTrigger EventName="ChangeLight" Target="{Binding}">
        <mvvm:GotoVisualStateAction />
    </mvvm:ViewModelTrigger>
</i:Interaction.Triggers>
```

It requires two properties - the target (typically the DataContext which is the ViewModel), and an event name - the event must be a public event with either no parameters (Action) or a single object parameter (Action). If a parameter is passed, it will be passed along to the action(s) associated with the trigger.

In the above example, I am also using a modified version of Blend's GotoStateAction - I built a derived version which doesn't require you to set the State property - if you don't set it, then it uses the passed parameter. So, in this case I can drive the specific visual state by passing it in my event from the View Model. Here's an example:

```csharp
public sealed class MainViewModel
{
    private Timer _timer;
    private StoplightState _lightState;

    public event Action<object> ChangeLight;

    public MainViewModel()
    {
        _timer = new Timer(OnChangeLight, null, 1000, 500);
    }

    private void OnChangeLight(object state)
    {
        switch (_lightState)
        {
            case StoplightState.Red:
                _lightState = StoplightState.Green;
                break;
            case StoplightState.Green:
                _lightState = StoplightState.Blue;
                break;
            case StoplightState.Blue:
                _lightState = StoplightState.Yellow;
                break;
            case StoplightState.Yellow:
                _lightState = StoplightState.Red;
                break;
        }

        if (ChangeLight != null) ChangeLight(_lightState.ToString());
    }
}
```

Here we use a timer to drive the state, but it could be based on logic, data received, etc. as well. When we raise the `ChangeLight` event, it is received by the View which in turn, uses the Visual State Manager to change our visualization to represent the new state. In our example here we show a series of blinking rectangles.

For this simple case here, we could also use a property change notification to drive the same visual state action. If I wanted to do that, I'd use the `[BindingTrigger](https://github.com/markjulmar/mvvmhelpers/blob/master/Julmar.Wpf.Helpers/Julmar.Wpf.Behaviors/Triggers/BindingTrigger.cs)`.

[Here's the full sample](/samples/GotoStateTest.zip) which includes both the `BindingTrigger` and `ViewModelTrigger` versions.
