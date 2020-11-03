---
title: Implementing disposable components
date: 2006-03-22T00:00:00.0000000
layout: post
categories: dotnet
tags: dotnet
---

A few months ago, I ran into a really weird bug with the **System.Net.NetworkInformation.Ping** class.  I was using it to monitor the network connectivity to a server and as my program ran, it appeared to leak handles.  I was calling **Dispose** (as I should) but it was still appeared to be leaking handles when called over and over. 

Here's the basic code I was running (simplified for this example):

```csharp
Ping icmp = new Ping();  
icmp.PingCompleted += delegate(object sender, PingCompletedEventArgs e)  
{  
      PingReply reply = e.Reply;  
      Console.WriteLine("Address: {0} - {1}", url, reply.RoundtripTime);  
      icmp.Dispose(); // Get rid of resources
};  
  
icmp.SendAsync(url, 100);  
  
// Continue doing work
```

The code looked ok to me, so I started looking a little deeper.  In reality, I found that if I waited long enough, the GC process was cleaning up the unmanaged handles using the SafeHandle class, but I was confused because Dispose should have done that for me. When I looked at the ping class using Reflector, the problem became obvious and it's a warning to anyone building components that need to free up non-memory related resources.

In order to integrate with the Windows Forms and ASP.NET designer, the **Ping** component extends **System.ComponentModel.Component**.  This provides the design-time integration with Visual Studio .NET and it also provides some basic plumbing that used to cleanup the component - specifically it implements IDisposable for you and provides a nice virtual **Dispose** method which you are supposed to override and free your resources.  The code follows Microsoft's [IDisposable](http://msdn.microsoft.com/en-us/library/system.idisposable(v=vs.110).aspx) pattern exactly - providing the IDisposable.Dispose method which delegates to an internal **virtual void Dispose(bool isDisposing)** method which is the method you should override.

The **Ping** component uses an internal socket and the ICMP W2K support under the covers to do its work and this socket needs to be cleaned up.  So, the author implemented **IDisposable** to indicate this -

```csharp
public class Ping : Component, IDisposable  
{  
    private void InternalDispose()  
    {  
        if (!disposed)  
        {  
             // Cleanup socket and/or ICMP handle resources..  
        }  
    }  

    IDisposable.Dispose()  
    {  
        this.InternalDispose();  
    }  
}
```

Note how the author used an explicit interface implementation for the **Dispose** method.  This means we will need to cast the object to an **IDisposable** interface in order to call the method (something I'm not doing above).  In fact, it won't even show up as a callable method inside VS.NET.  There is nothing wrong with this implementation (except of course it doesn't follow Microsoft's guidelines) until you add in the derivation from **Component**.  If we add it's implementation into the mix, and expand it out I get something like:

```csharp
public class Ping : Component, IDisposable  
{  
    private void InternalDispose()  
    {  
        if (!disposed)  
        {  
            // Cleanup socket and/or ICMP handle resources..  
        }
    }  
  
    IDisposable.Dispose()  
    {  
        this.InternalDispose();  
    }  
  
    public void Dispose()  
    {  
        this.Dispose(true);  
        GC.SupressFinalize(this);  
    }  
  
    protected virtual void Dispose(bool disposing)  
    {  
        // Remove object from component container
    }  
}
```

See the problem?  I now have a public **Dispose** method available directly - and intellisense will show that.  This is the one I was calling my code when my async call was finished.  The problem is that it wasn't actually disposing the unmanaged resources - it was running the **Component.Dispose** code.

I need to back up and make another point on this.  I wouldn't have even seen the problem if I had been doing things synchronously.  In that case, I would have likely doing a **using** statement:

```csharp
using (Ping icmp = new Ping())  
{  
    PingReply reply = icmp.Send(url, 100);  
    if (reply.Status == IPStatus.Success)  
    {  
        Console.WriteLine("Address: {0} - {1}", url, reply.RoundtripTime);  
    }  
}
```

In this case, the C# compiler is nice enough to dispose of the object for me - and it does it by casting the object to **IDisposable**.  So, the generated code would _really_ look like:

```csharp
Ping icmp = new Ping();  
try  
{  
    PingReply reply = icmp.Send(url, 100);  
    if (reply.Status == IPStatus.Success)  
    {  
        Console.WriteLine("Address: {0} - {1}", url, reply.RoundtripTime);  
    }  
}  
finally  
{  
    ((IDisposable)icmp).Dispose();  
}
```

So, this would end up calling the correct implementation.  It was only because I was calling **Dispose** directly that I had a problem.  If the author had followed the IDisposable guidelines, this problem would have been found immediately because the C# compiler would have spit out a warning that **public void Dispose()** is hiding a base class implementation - cluing the author in that they need to hook into the base class implementation.  So, how did this happen?  My guess is that originally the **Ping** class didn't extend Component.  That derivation was added later in order to provide for design-time support.

If you are building components yourself, don't fall into this trap!  Always, always, always use Microsoft's stated guidelines - here's a simple example for those not familiar with it:

```csharp
public class MyResource: IDisposable  
{  
    // Track whether Dispose has been called.  
    protected bool disposed = false;  

    // Implement IDisposable.  
    // Do not make this method virtual.  
    // A derived class should not be able to override this method.  
    public void Dispose()  
    {  
        Dispose(true);  
        // This object will be cleaned up by the Dispose method.  
        // Therefore, you should call GC.SupressFinalize to  
        // take this object off the finalization queue  
        // and prevent finalization code for this object  
        // from executing a second time.  
        GC.SuppressFinalize(this);  
    }

    // Dispose(bool disposing) executes in two distinct scenarios.  
    // If disposing equals true, the method has been called directly  
    // or indirectly by a user's code. Managed and unmanaged resources  
    // can be disposed.  
    // If disposing equals false, the method has been called by the  
    // runtime from inside the finalizer and you should not reference  
    // other objects. Only unmanaged resources can be disposed.  
    protected virtual void Dispose(bool disposing)  
    {  
        // Check to see if Dispose has already been called.  
        if(!this.disposed)  
        {  
            // If disposing equals true, dispose all managed  
            // and unmanaged resources.  
            if(disposing)  
            {  
                // Dispose all managed resources here.  
            }  

            // Call the appropriate methods to clean up  
            // unmanaged resources here.  
        }  
        disposed = true;  
   }  
}  
```

There's a bunch more information on this topic is section 9.3 of the [Framework Design Guidelines](http://www.amazon.com/gp/product/0321246756/sr=8-1/qid=1143055197/ref=pd_bbs_1/002-4294371-8914451?%5Fencoding=UTF8) - a must read for anyone building class libraries or reusable components.
