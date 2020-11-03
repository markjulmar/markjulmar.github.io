---
title: Problem with MEF in Windows 8 style apps (Metro)
date: 2012-08-24T00:00:00.0000000
layout: post
categories: mvvm
tags: mvvm dotnet
---

I noticed when I use NuGet to add MVVMHelpers, the app refuses to start.  I get the following error:

> "Unable to activate Windows Store app xxx, The activation request failed with error 'The app didn't start'."

![](/images/vserr_metro.png "Metro App Error")

Wow. Totally unhelpful - even chasing down the event logs didn't really help much other than to show me a lot of weird errors get logged from Metro apps about tiles!

After experimenting a bit, it turns out the fix is quite simple.  MVVMHelpers relies on MEF for composition and dependency injection.  The MEF package (**Microsoft.Composition**) which also gets included, adds an **app.config** with a bunch of assembly binding instructions for version redirection - this is what is causing the failure.  I presume these are for the desktop version of .NET and just don't work with Windows 8 apps. Here's what got added to mine:

```xml
<runtime>
    <assemblyBinding xmlns="urn:schemas-microsoft-com:asm.v1">
        <dependentAssembly>
            <assemblyIdentity name="System.Composition.TypedParts" publicKeyToken="b03f5f7f11d50a3a" culture="neutral"/>
            <bindingRedirect oldVersion="0.0.0.0-1.0.15.0" newVersion="1.0.15.0"/>
        </dependentAssembly>
        <dependentAssembly>
            <assemblyIdentity name="System.Composition.Hosting" publicKeyToken="b03f5f7f11d50a3a" culture="neutral"/>
            <bindingRedirect oldVersion="0.0.0.0-1.0.15.0" newVersion="1.0.15.0"/>
        </dependentAssembly>
        <dependentAssembly>
            <assemblyIdentity name="System.Composition.Runtime" publicKeyToken="b03f5f7f11d50a3a" culture="neutral"/>
            <bindingRedirect oldVersion="0.0.0.0-1.0.15.0" newVersion="1.0.15.0"/>
        </dependentAssembly>
        <dependentAssembly>
            <assemblyIdentity name="System.Composition.AttributedModel" publicKeyToken="b03f5f7f11d50a3a" culture="neutral"/>
            <bindingRedirect oldVersion="0.0.0.0-1.0.15.0" newVersion="1.0.15.0"/>
        </dependentAssembly>
    </assemblyBinding>
</runtime>
```

Remove the **app.config** to solve the problem. I hope that saves somebody some time and grief! Happy coding!
