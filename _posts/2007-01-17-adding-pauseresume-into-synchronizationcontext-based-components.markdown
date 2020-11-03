---
title: Adding Pause/Resume into SynchronizationContext based components
date: 2007-01-17T00:00:00.0000000
layout: post
categories: dotnet
tags: dotnet
---

About a year ago, I blogged on using the .NET 2.0 facility **SynchronizationContext** to create thread-aware components which could be hosted and would "do the right thing" for callbacks.

Recently, I got an email from a reader who wanted to use that paradigm, but wanted to be able to suspend the operation for some period of time and then resume it later on. He asked if there was an easy way to modify the sample I presented, and it turns out that there is.

It essentially involves three steps:

1. In the `AsyncStateData` structure, add a `ManualResetEvent` object. This will be used to trigger the pause/resume behavior. It must be initially set as signaled.

1. In the actual asynchronous operation loop, add the event into the loop condition by calling `event.WaitOne()`. Make sure you have released any locks and the code can safely be halted at that point in the logic otherwise you may create deadlock scenarios and other difficult debugging issues.

1. Add a `Pause` and `Continue` method into the class which signals and resets the event.

Modifying my previous code example, here's what I come up with:

## Step 1: Add the event to AsyncStateDate

```csharp
class AsyncStateData
{
    private ManualResetEvent pauseEvent = new ManualResetEvent(true);
    ...
}
```

## Step 2: Add event into the loop at a safe point

```csharp
public partial class Calculator
{
    private void InternalCalculatePi(int digits, AsyncStateData asyncData)
    {
       int completedDigits = 0;
       for (; !asyncData.canceled && pauseEvent.WaitOne(-1, true) && completedDigits < digits; completedDigits++)
       {
           ...
       }
    }
    ...
}
```

## Step 3: Add a Pause and Continue API

```csharp
public partial class Calculator
{
    public void PauseAsync(object asyncTask)
    {
        AsyncStateData asyncData = asyncTask as AsyncStateData;
        if (asyncData != null && asyncData.running == true)
            asyncData.pauseEvent.Reset();
    }

    public void ContinueAsync(object asyncTask)
    {
        AsyncStateData asyncData = asyncTask as AsyncStateData;
        if (asyncData != null && asyncData.running == true)
            asyncData.pauseEvent.Set();
    }
    ...
}
```

There are certainly other ways to do this as well, for example, create a dedicated Thread object instead of using an Async delegate and then call `Suspend` on the thread (and `Resume` to continue). That can be dangerous if you take locks while running your async operation so the `ManualResetEvent` is probably a better solution from that perspective (since you are blocking in a known location vs. just suspending the thread arbitrarily and potentially holding onto shared resources.
