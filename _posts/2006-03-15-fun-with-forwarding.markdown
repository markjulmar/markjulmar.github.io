---
title: Fun with forwarding..
date: 2006-03-15T00:00:00.0000000
layout: post
categories: tapi
tags: tapi
---

Forwarding lines with ATAPI.NET is simple and easy (assuming, of course, that the underlying TSP supports it).

The first step is knowing whether a given line device even _supports_ forwarding.  This is trivial:

```csharp
TapiManager mgr = new TapiManager("ForwardingTest");

foreach (TapiLine line in mgr.Lines)  
{ if (line.Capabilities.SupportsForwarding)  
   {  
      Console.WriteLine("Line {0} supports forwarding!", line.Name);  
   }  
}
```

Once we've identified a specific line, we can look at each address and get more information such as the _types_ of forwarding supported.  For example, we might be able to forward to different numbers based on specific conditions such as whether the call goes unanswered vs. whether the address is in use and returning a busy signal.  We might also be able to forward specific inbound callers (very useful to get rid of your bosses calls).  We can get this information from the **Capabilities** of the **TapiAddress** object:

```csharp
foreach (TapiAddress addr in line.Addresses)  
{
    Console.WriteLine("Forwarding modes supported on {0} are {1}", addr.Address, addr.Capabilities.SupportedForwardingModes);  
}
```

We can also retrieve any existing forwarding information through the **Status** of the **TapiAddress**:

```csharp
foreach (ForwardInfo fwd in addr.Status.ForwardingInformation)
    Console.WriteLine("t{0} to {1}:{2}", fwd.ForwardMode, fwd.DestinationAddressType, fwd.DestinationAddress);
```

This outputs: "Unconditional to PhoneNumber:1234" on a forwarded line I setup.

Finally, the big question is how to change the forwarding information, this is pretty easy as well.  You can set forwarding information on two levels, the entire line (which impacts all addresses), or a specific address.  This is done through two methods present on both **TapiAddress** and **TapiLine** which are **Forward** and **CancelForward**.  So, to cancel all forwarding in effect on every line we could do the following:

```csharp
Console.WriteLine("Canceling all forwards:");  
foreach (TapiLine line in mgr.Lines)  
{
    if (line.Capabilities.SupportsForwarding)
    {
        try
        {  
            line.CancelForward();  
        }
        catch (TapiException ex)
        {
            Console.WriteLine("{0} - {1}", line, ex.Message);  
        }
    }  
}
```

Or, to setup the forwarding as above, I can issue a call to the **Forward** method:

```csharp
ForwardInfo[] fwdInfo = new ForwardInfo[] { new ForwardInfo(ForwardingMode.Unconditional, 0, "1234") };

foreach (TapiLine line in mgr.Lines)  
{
    if (line.Capabilities.SupportsForwarding)  
    {
        try
        {  
            line.Forward(fwdInfo, 5, null);  
        }
        catch (TapiException ex)
        {
            Console.WriteLine("{0} - {1}", line.Name, ex.Message);  
        }  
    }  
}
```

The **ForwardInfo** class describes a single forwarding instruction and you pass an array of these info the **Forward** method to indicate how things are to be managed.  Exceptions need to be handled because the TAPI service provider might not allow the particular forwarding at this point in time, or the destination might not be allowed, etc.

Under the covers this will issue a **lineForward** request with a **LINEFORWARDLIST** setup for each of the **ForwardInfo** structures.
