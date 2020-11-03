---
title: Adding to an ObservableCollection from a background thread
date: 2009-04-01T00:00:00.0000000
layout: post
categories: dotnet
tags: dotnet
---

One annoying thing about `ObservableCollection<T>` is that it doesn't support modifications from background threads.  That is to say, the `CollectionChanged` notification doesn't marshal to the proper thread when it is raised and it causes the exception **"This type of CollectionView does not support changes to its SourceCollection from a thread different from the Dispatcher thread"**. 

I ran into this problem a while back and searched out to see if someone else had solved it. I found a solution from Tamir Khason (a fellow WPF disciple), he wrote a [thread-safe version](http://blogs.microsoft.co.il/blogs/tamir/archive/2007/04/22/Thread-safe-observable-collection.aspx), but I didn't want to add the locking into the collection itself (I want to manage it at a higher level myself).  Really, all I want is to do the notification on the proper (Dispatcher) thread.

It turns out to be pretty easy, here's my solution:

```csharp
public class MTObservableCollection<T> : ObservableCollection<T>  
{  
    public override event NotifyCollectionChangedEventHandler CollectionChanged;  
    protected override void OnCollectionChanged(NotifyCollectionChangedEventArgs e)  
    {  
        var eh = CollectionChanged;  
        if (eh != null)  
        {  
            Dispatcher dispatcher \= (from NotifyCollectionChangedEventHandler nh in eh.GetInvocationList()  
                 let dpo = nh.Target as DispatcherObject  
                 where dpo != null  
                 select dpo.Dispatcher).FirstOrDefault();  

            if (dispatcher != null && dispatcher.CheckAccess() == false)  
            {  
                dispatcher.Invoke(DispatcherPriority.DataBind, (Action)(() => OnCollectionChanged(e)));  
            }  
            else
            {  
                foreach (NotifyCollectionChangedEventHandler nh in eh.GetInvocationList())  
                    nh.Invoke(this, e);  
            }  
        }
    }
}
```
