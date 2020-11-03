---
title: INotifyPropertyChanged in .NET 4.5
date: 2012-03-01T00:00:00.0000000
layout: post
categories: dotnet
tags: dotnet
---

Implementing `INotifyPropertyChanged` in your XAML-based application has always been a point of hot debate. To date there are a variety of techniques - from the original (and fastest) "magic" string indicating the property name to the much slower, but compiler-checked Lambda expression tree implementation which provides refactoring safety. Both styles are included in my MVVM library ([http://mvvmhelpers.codeplex.com](http://mvvmhelpers.codeplex.com)). Trying to keep the performance of a raw string-based approach, but get the compile-time safety of the expression tree approach has been the goal of many WPF and Silverlight devs for the past few years. We're getting closer...

.NET 4.5 (included in VS11 beta this week) brings a new option to the table in the form of the new `[CallerMemberName]` attribute. I saw this in Ander's talk at //BUILD last year and immediately thought of `INotifyPropertyChanged`. Essentially, this attribute is applied to an input parameter on a method and the CLR will supply the calling method name as the parameter's value. As an example, here is a sample `INotifyPropertyChanged` implementation using this new feature:

```csharp
public class Person : INotifyPropertyChanged
{
    private string _name;
    public string Name
    {
        get { return _name; }
        set
        {
            if (_name != value)
            {
                _name = value;
                RaisePropertyChanged(); // note no parameter here!
            }
        }
    }

    public event PropertyChangedEventHandler PropertyChanged = delegate { };
    private void RaisePropertyChanged([CallerMemberName] string propertyName = "")
    {
        PropertyChanged(this, new PropertyChangedEventArgs(propertyName));
    }
}
```

Stopping in the debugger, you can see the value of the `propertyName` parameter: ![](/images/INPCPicture.png "INPCPicture")

The performance (at least of .NET 4.5 beta) is in between the unsafe magic string approach and the nice lambda approach - but this is perfectly refactor safe so it's a nice new way to improve the safety of our code while getting closer to the performance of magic strings with our property change notifications.

This solution also provides for the "all properties have changed" notification by supplying the string - or passing null as the parameter.  If you specifically provide a parameter, then it overrides the `[CallerMemberName]` attribute and you get that string instead.  So, to raise an "all properties have changed" notification you could do this:

```csharp
RaisePropertyChanged(""); // or RaisePropertyChanged(null);
```

and to raise a different property altogether you can just pass the magic string as you always have -

```csharp
// Callback from some long-running operation
void OnCalculateSomeValueCompleted()
{
    RaisePropertyChanged("CalculatedValue");
}
```

It's not the perfect solution however - as you can see above, we still have to use field-backed properties with this approach.  It would be so much nicer to have some compiler support here which calls the `RaisePropertyChanged` method from an auto-property.  This is done by some frameworks today using an AOP approach, but I'd like to see it with the native toolset.

This is a good step in the right direction though - thanks MSFT!

> **EDIT:**
>
> interestingly enough, I noticed that the latest MSDN example of `INotifyPropertyChanged` uses this exact approach - see here: [http://msdn.microsoft.com/en-us/library/ms229614(v=vs.110).aspx](http://msdn.microsoft.com/en-us/library/ms229614(v=vs.110).aspx "How to: Implement the INotifyPropertyChanged Interface")
