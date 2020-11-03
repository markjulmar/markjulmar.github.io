---
title: Wrapping the TAPI 3.0 API with C++/CLI
date: 2006-03-03T00:00:00.0000000
layout: post
categories: tapi
tags: tapi
---

### New announcement - New .NET library for TAPI 3.x

Recently, I released the [ATAPI.NET project](https://github.com/markjulmar/atapi.net) into the wild for people to be able to easily code up .NET applications that utilize TAPI.  That code is based on the C API exported from **TAPI 2.1** and so uses the older form of TAPI programming.

With Windows 2000, Microsoft released a COM version of TAPI - dubbed **TAPI 3.0** that was designed to allow VB developers to access the telephony API.  However, as I noted in my previous post, Microsoft claims that the object model is too complex and that there are significant issues with accessing it from managed code.

.NET has a built-in facility to access COM objects, it creates .NET "runtime-callable" wrappers (called RCW's) around the COM interfaces and then allows you to call the COM object as if it were a .NET object.  That's great until you need to mesh the COM style of cleanup (AddRef/Release) with the .NET style (GC).  Essentially, the COM object doesn't get released until the managed wrapper gets collected.  In addition, if you have multiple wrappers for a single interface (TAPI returns the same interface through different methods), the interface won't be released until **all** the wrappers are collected.

Sometimes, this isn't really a problem - so we keep a COM interface alive a little longer than normal.  In TAPI's case however it can be a huge problem because some TAPI service providers will not let you create new calls unless the existing call interface has been released - even if it isn't connected to anything.

So, two options exist today:

1) Don't use TAPI 3.x - use TAPI 2.x where you have more control over the underlying TAPI call handle and can call `lineDeallocate` yourself.  In .NET this means using some wrapper like ATAPI.NET.  If you don't need terminal and stream support, this is fine.  This is what the newsgroups typically will recommend to people.

2) Call `Marshal.ReleaseCOMObject` on every outstanding interface.  Just plain ugly and extremely error-prone.

Now, a new option exists.  I've been in conversations with Matthias Moetje - one of the TAPI MVPs and went back to my original TAPI3 wrapper which was written in MC++ (yuck) and ported it to C++/CLI (which, BTW, is very cool).  It has almost everything supported except agents and call center support.  It solves the above problem in a couple of steps:

1. It guarantees that the same interface will match 1:1 with a managed object as long as the object hasn't been collected yet.  This means the object will only have one holding managed object.  
1. It implements `IDisposable` on most of the wrappers allowing the client to get rid of interfaces immediately.  
1. It "auto-disposes" calls when they hit the disconnected state.  This can be turned on or off depending on need through the `TTapi.AutoDestroyCall`s flag

Instead of a `Register` function, each address has an `Open` and `Monitor` method and a `Close` method. Since you have to pass the address in anyway, I moved the function to that level and made it distinct rather than having you pass in two booleans to indicate intent.

There are some shorthand functions: for example, the `TCall` object (which represents `ITBasicCallControl`, `ITCallInfo`, et.al.) has a `SelectDefaultTerminals` method which enumerates the streams and hooks up the default static/video terminals for each stream - basically the code that was in every TAPI sample provided by MS. There's also a `FindTerminal` method on the `TStream` class which will locate a terminal of the proper media type/direction if it exists.

The library and all source code is under the MIT open source license and available on GitHub [here](https://github.com/markjulmar/itapi3).

The library itself follows the TAPI3 interfaces pretty closely - except it combines the various control interfaces together.  So for example:

| .NET Class | Wrapped Interfaces |
|-------|------------|
| `TTapi` | `ITTapi`, `ITTapi2` |
| `TAddress` | `ITAddress`, `ITAddress2`, `ITAddressCapabilities`, `ITAddressTranslation`, `ITLegacyAddressMediaControl`, `ITLegacyAddressMediaControl2`, `ITMediaSupport`, `ITTerminalSupport` |
| `TCall` | `ITCallInfo`, `ITBasicCallControl`, `ITBasicCallControl2`, `ITStreamControl`, `ITLegacyCallMediaControl`, `ITLegacyCallMediaControl2` |

In each case, the same properties and methods are exposed by the reference class and it will do the underlying cast necessary to get to the functionality.  The downside of this, is that if the interface isn't supported, the library must throw an exception - in this case a TapiException which holds a string message indicating the failure and an error code (the `HRESULT`).  The `TTapi` class exposes the **TE_xxx** events you generally registered a sink for.

I didn't bother to encapsulate the DirectShow library - instead each of the TAPI wrappers provides a QueryInterface method which you can use to get to an interface if necessary.  So, for example, if you want to get to the venerable `IVideoWindow` interface from DirectShow, you can include an interop reference to Quartz.DLL and then do the following on the `TTerminal` object:

```csharp
IVideoWindow videoWindow = thisTerminal.QueryInterface(typeof(IVideoWindow)) as IVideoWindow;  
if (videoWindow != null)
{
    ...
}
```

So, here's an example of using it - I think it's much cleaner that the comparable RCW code, and it's thread-aware and works around the general problem of TAPI3 under .NET through a `Dispose` mechanism and the `TTapi.AutoDestroyCalls` flag.

```csharp
using System;
using System.Collections.Generic;
using System.Text;
using JulMar.Tapi3;

namespace TestTapi
{
    class Program
    {
        static void Main(string[] args)
        {
            TTapi tapi = new TTapi();
            TCall call = null;
            TAddress modemAddr = null;

            tapi.Initialize();

            tapi.TE_CALLNOTIFICATION += delegate(object sender, TapiCallNotificationEventArgs e)
            {
                Console.WriteLine("New call {0} detected from {1}", e.Call.ToString(), e.Event);
            };

            tapi.TE_CALLSTATE += delegate(object sender, TapiCallStateEventArgs e)
            {
                Console.WriteLine("{0}:{4} has changed state to {1} due to {2} - current={3}:{5}", e.Call, e.State,
                    e.Cause, e.Call == call, e.Call.GetHashCode(), (call != null) ? call.GetHashCode() : 0);
                if (e.State == CALL_STATE.CS_INPROGRESS && e.Call == call)
                {
                    Console.WriteLine("Dropping call");
                    e.Call.Disconnect(DISCONNECT_CODE.DC_NORMAL);
                }
            };

            foreach (TAddress addr in tapi.Addresses)
            {
                if (String.Compare(addr.ServiceProviderName, "unimdm.tsp", true) == 0 &&
                    addr.QueryMediaType(TAPIMEDIATYPES.AUDIO))
                    modemAddr = addr;
            }

            if (modemAddr != null)
            {
                Console.WriteLine("{0} = {1} ({3}) [{2}]", modemAddr.AddressName, modemAddr.State,
                    modemAddr.ServiceProviderName, modemAddr.DialableAddress);
                modemAddr.Monitor(TAPIMEDIATYPES.AUDIO);
                ConsoleKey ki = ConsoleKey.A;

                while (ki != ConsoleKey.Q)
                {
                    // Flip the auto-destroy flag if (ki == ConsoleKey.D)   
                    {
                        tapi.AutoDestroyCalls = !tapi.AutoDestroyCalls;
                        Console.WriteLine("Set AutoDestroy to {0}", tapi.AutoDestroyCalls);
                    } // List existing calls  
                    else if (ki == ConsoleKey.L)
                    {
                        foreach (TCall _call in modemAddr.EnumerateCalls())
                        {
                            Console.WriteLine("Existing call found: {0}:{1}", _call, _call.GetHashCode());
                            _call.Dispose(); // Go ahead and dump it }  
                        } // Create a new call else {  

                        call = modemAddr.CreateCall("5551213", LINEADDRESSTYPES.PhoneNumber, TAPIMEDIATYPES.DATAMODEM);
                        Console.WriteLine("Created new call {0}:{1}", call, call.GetHashCode());
                        try
                        {
                            // This will fail if existing call interface is still around (i.e. not disposed) call.Connect(false);  
                        }
                        catch (TapiException ex)
                        {
                            Console.WriteLine(ex.Message);
                        }
                    }

                    Console.WriteLine("Press a key to try another call.. Q to quit");
                    ki = Console.ReadKey().Key;
                }
            }

            // This will destroy any outstanding interfaces
            tapi.Shutdown();

            // Call should be disposed here.. state will be CS_UNKNOWN
            if (call != null)
                Console.WriteLine("{0} {1}", call, call.CallState);  
        }
    }
}
```
