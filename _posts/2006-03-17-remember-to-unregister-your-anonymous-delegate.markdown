---
title: Remember to unregister your anonymous delegate!
date: 2006-03-17T00:00:00.0000000
layout: post
categories: dotnet
tags: dotnet
---

I love anonymous delegates - I think they are extremely useful and allow me to solve some problems in very elegant ways.  However, once you really get into them, you start to see the dark side of anonymous delegates and that is unregistration.

Here's the basic problem: when binding an instance delegate to an event to handle some activity, the delegate will cache off the instance reference - thereby keeping the reference alive.  So for example, if I had a form which wanted to process some activity from an object:

```csharp
class Publisher
{
    public event EventHandler OnEvent;
        ...
}

class MainForm : Form
{
    Publisher _pub = new Publisher();

    void ActivateChildForm()
    {
        ChildForm f = new ChildForm(_pub);
        f.Show();
    }
}

class ChildForm : Form
{
    public ChildForm(Publisher pub)
    {
        pub.OnEvent += ProcessEvent;
    }

    void ProcessEvent(object sender, EventArgs e)
    {
        listBox1.Items.Add("Event was fired!");
    }
}
```

When the ChildForm instance is closed, the form won't be collected because the Publisher (_pub) is holding a reference to it.  This is easily fixed by adding some code into the FormClosing event:

```csharp
class ChildForm : Form
{
    Publisher _pub;

    public ChildForm(Publisher pub)
    {
        _pub = pub;
        _pub.OnEvent += ProcessEvent;
    }

    void FormClosing(object sender, FormClosingEventArgs e)
    {
        _pub.OnEvent -= ProcessEvent;
    }
}
```

Now, the form will be cleaned up when it's closed.  This is pretty standard stuff, and most people that have been using .NET for a while know all this.  Here's the rub: with .NET 2.0, we can simplify the code using anonymous delegates which are really helpful for these single-line processing event handling functions.  So, I could recode my handler as:

```csharp
class ChildForm : Form  
{  
    public ChildForm(Publisher pub)
    {  
        pub.OnEvent += delegate { listBox1.Items.Add("Event was fired!"); }  
        this.OnClosing += delegate { pub.OnEvent -= ???? }
    }  
}
```

The issue is that I don't have a reference to the delegate as it's typed.  Under the covers, the C# compiler has generated a temporary function (or possibly even pulled it out to a separate inner class) and there's no way for me to get to the underlying function.  So, what can I do?  Well, the easiest thing to do is to save off the function:

```csharp
class ChildForm : Form  
{  
    public ChildForm(Publisher pub)
    {  
        EventHandler eh = delegate { listBox1.Items.Add("Event was fired!"); }  
        pub.OnEvent += eh; this.OnClosing += delegate { pub.OnEvent -= eh; } 
    }  
}
```

Now my code will function properly -- and is significantly reduced in size.  Of course, I've lost the benefit of being able to hook up events through VS.NET because it always generates separate functions and I would need to cache off the Publisher instance as well as my delegate in that case. 

So, rule #1, always unregister the event when you are finished.  Rule #2, remember that anonymous delegates may be keep your instance alive so unregister them as well, _unless_ the event is to be hooked up throughout the lifetime of the application.
