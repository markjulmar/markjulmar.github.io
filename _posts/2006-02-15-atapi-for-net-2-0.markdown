---
title: ATAPI for .NET 2.0
date: 2006-02-15T00:00:00.0000000
layout: post
categories: tapi
tags: tapi
---

Ok, ok so it's been a while.  I've been very busy with two tasks -- first, I spent the last few weeks doing a Guerrilla .NET for **DevelopMentor** with Rich Blewitt.  We had a blast together and it was great to hang out with him.  I also spent a day sitting in on DM's new C++/CLI class being taught by the very capable Marcus Heege - incredible stuff which every C++ guy on the Microsoft platform should get into.

The other thing I've been working with is resurrecting an old project of mine - ATAPI which was originally setup to wrap the TAPI 2.x API in an "easy to use" set of C++ classes.  I'd ported it to .NET a few years ago but was not really happy with the results.  I had the chance to revisit it because of a client's requirement to integrate TAPI into their .NET platform code.  So, I spent a couple of weeks working on the codebase again under .NET 2.0 and this time around I'm pretty pleased with the architecture.  I wanted something very easy to use, and I think I've achieved that even though it isn't a complete wrapper.

For example, to walk through all the lines and dump out the device classes available - you can simply do this:

```csharp
using System;  
using System.Collections.Generic;  
using System.Text;  
using JulMar.Atapi;

namespace EnumDevices  
{  
    class Program  
    {  
        static void Main(string[] args)  
        {  
            TapiManager mgr = new TapiManager("EnumDevices");  
            mgr.Initialize(); // Start up Tapi

            foreach (TapiLine line in mgr.Lines)  
            {  
                foreach (string s in line.Capabilities.AvailableDeviceClasses)  
                    Console.WriteLine("{0} - {1}", line.Name, s);  
            }
            mgr.Shutdown();  
        }  
    }  
}
```

_So much easier than the C/C++ version!_

### TAPI 3.0

So.. why not use the TAPI3 COM API you ask?  Well, as it turns out, it doesn't work that well with the runtime-callable wrapper (RCW) infrastructure in .NET -- check out [this support article](http://support.microsoft.com/kb/841712/en-us) where Microsoft basically says "TAPI3 is too complicated".. like I needed someone to tell me that!

The ATAPI.NET source code and samples are available [here](https://github.com/markjulmar/atapi.net).

enjoy.
