---
title: LINQ is just freaking cool
date: 2009-03-05T00:00:00.0000000
layout: post
categories: dotnet
tags: dotnet
---

A couple of weeks ago, Jason Whittington (a fellow instructor at **Developmentor** and I were doing a talk on asynchronous programming and we started with a very simple example of the APM -- using two loops to create and then consume the `IAsyncResult` work:

```csharp
static void TwoLoopMain(string[] args)  
{  
   Queue<IAsyncResult> ars = new Queue<IAsyncResult>();  
   Func<int,int,int> mathProc = Multiply;

    for (int i = 1; i <= 20; i++)  
    {  
        for (int j = 1; j <= 20; j++)  
            ars.Enqueue(mathProc.BeginInvoke(i, j, null, null));  
    }  
    for (int i = 1; i <= 20; i++)  
    {  
        for (int j = 1; j <= 20; j++)  
        {  
            int result = mathProc.EndInvoke(ars.Dequeue());  
            Console.SetCursorPosition(i * 3, j);  
            Console.Write(result);  
        }  
    }  
}  
```  

We had already presented a talk on LINQ earlier in the week and so I thought we might be able to do the above in a single expression with LINQ - kind of a challenge .. here's what we came up with:

```csharp
static void LinqTest()  
{  
    Func<int,int,int> mathProc = Multiply;  
  
    (from i in Enumerable.Range(1, 20)  
     from j in Enumerable.Range(1, 20)  
     select new { i, j, ar = mathProc.BeginInvoke(i, j, null, null) })  
    .ToList()  
    .ForEach(e => {  
         int result = mathProc.EndInvoke(e.ar);  
            Console.SetCursorPosition(e.i * 3, e.j);  
            Console.Write(result);  
    });  
}
```

It's not easily readable (and so I wouldn't promote this for production code), but it's way cool and an example of how LINQ (and functional programming in general) is really changing the way programmers think about code.. just freaking cool..
