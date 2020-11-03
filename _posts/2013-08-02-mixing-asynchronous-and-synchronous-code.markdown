---
title: Mixing asynchronous and synchronous code
date: 2013-08-02T00:00:00.0000000
layout: post
categories: dotnet
tags: dotnet
---

I had a question from a former student today related to a deadlock he was encountering in his Windows Store application. He had an interface which was already in use (and therefore could not be changed) which had to call a method that performs I/O. In WSAs (Windows Store Apps) all I/O is now asynchronous - so he used the async/await keyword which is the normal approach here. The problem was that he had to block waiting on the results due to the interface design. Here's a similar block of code to what he originally sent me (simplified for the purposes of this example):

```csharp
private void OnButtonClick(object sender, RoutedEventArgs e)
{
    Log("Starting test..");

    string result = PleaseWaitForMe();
    Log(result);
}

private string PleaseWaitForMe()
{
    bool available = IsWebsiteAvailable("www.microsoft.com").Result;
    return (available) ? "Online" : "Offline";
}

public async Task IsWebsiteAvailable(string address)
{
    using (var tcpClient = new StreamSocket())
    {
        await tcpClient.ConnectAsync(new HostName(address), "http");
        return true;
    }
}
```

> **Note:**
>
> the Log method above is just a method to print something to the UI - check out the included sample to see what it does

In the above code, we have a button click handler which calls a method that blocks waiting on I/O. When you click the button, the app just freezes - exhibiting a classic deadlock. The problem in this case is that the async / await used in the **IsWebsiteAvailable** method kicks off the I/O request but then tries to harvest the results on the UI thread in order to complete the task. However, the UI thread is blocked waiting on the returned task - which is waiting to process the "finished" message on the same thread.

Now, in a perfect world, we would redesign this application to push the async I/O up the call stack so we could use async/await all the way through the chain. This code is actually flawed because it is blocking the UI thread deliberately and the last thing we really want to do is block the UI thread because that violates the "fast and fluid" mantra of mobile apps. In this instance, it was not possible to redesign the interface - so I recommended adding a new interface which returns a `Task<string>` instead of just a `string`.

However, let's say we just need to fix this problem in the most expedient way possible and an architecture/app redesign just isn't possible now. Enter `Task.Run`. This API was added with .NET 4.5 and it was introduced specifically to help with scenarios like this where we need to control where a task runs. The idea is to push the I/O work and it's results off onto a real worker thread (introducing a new thread which isn't ideal but will keep us from deadlocking). `Task.Factory.StartNew` is the older API but it always uses the current scheduler (which might be the UI scheduler which would be bad in this case) and it isn't aware of the inner task and so we'd have to _Unwrap_ the inner task to get to the results. `Task.Run` solves all of that. The simple fix here looks like this:

```csharp
public Task IsWebsiteAvailable(string address)
{
    return Task.Run(async () =>
    {
        using (var tcpClient = new StreamSocket())
        {
            await tcpClient.ConnectAsync(new HostName(address), "http");
            return true;
        }
    })
}
```

Notice that we've removed the async keyword from the method signature - we want the raw task to be returned now. Instead, we've moved it onto the lambda expression being passed into `Task.Run`. This ignores the current task scheduler and always uses the worker thread scheduler. So, the code to check the website availability will now execute on a threadpool thread (and so it will perform the continuation code executed after the await on the same threadpool thread). The UI thread will kick off the new task/thread and then block waiting for it to return the results.

This code works properly - in that the app doesn't deadlock, but keep in mind that it's possible that it might block the UI thread for a long period of time, eventually failing and throwing an exception (or being canceled if we passed in a cancellation token with timeout). However, when you are mixing async and synchronous code together these are the kinds of compromises you end up with.

If you want to play with the full sample, [here it is](/samples/MixingSynchAndAsync.WSA.TestApp.zip).
