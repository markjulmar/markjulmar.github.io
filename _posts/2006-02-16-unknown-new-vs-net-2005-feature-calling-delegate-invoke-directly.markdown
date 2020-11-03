---
title: Unknown new VS.NET 2005 feature - Calling Delegate.Invoke directly
date: 2006-02-16T00:00:00.0000000
layout: post
categories: dotnet
tags: dotnet
---

So, at the GNET2 last week, I found out that you can now call `Delegate.Invoke` instead of using the C# functor syntax.  This makes code much more readable IMO --

```csharp
delegate void MyCallback(string msg);  
...  
MyCallback callback;  
...  
callback += delegate(string msg) { Console.WriteLine(msg); };  
...  
callback.Invoke("Hello"); // instead of callback("Hello");
```

Very nice.
