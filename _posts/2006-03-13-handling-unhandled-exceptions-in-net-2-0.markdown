---
title: Handling Unhandled Exceptions in .NET 2.0
date: 2006-03-13T00:00:00.0000000
layout: post
categories: dotnet
tags: dotnet
---

I got hit with this problem from two separate clients last week - a .NET 1.1 application ported to .NET 2.0 is now terminating abruptly for no apparent reason.  Well, of course there's a reason - and it's that both applications had "hidden" exceptions being thrown in some background thread that weren't being caught.  Under .NET 1.1, the CLR would print any exceptions that occurred on threadpool threads to the console and then return the thread to the pool.  In addition, the CLR would silently eat any exception thrown on the finalizer thread (again printing the stack trace to the console).  Under .NET 2.0, the behavior has changed and it will cause the AppDomain to be unloaded (yikes!).  So, for example:

```csharp
class BadClass
{
    ~BadClass()
    {
        throw new Exception("I'm Bad");
    }
}

class BadClass
{
    static void Main()
    {
        for (;;)
            new BadClass();
    }
}
```

The above code would run forever under CLR 1.1, but will terminate immediately when run under CLR 2.0 with an unhandled exception.

```output
Unhandled Exception: System.Exception: I'm bad  
   at BadClass.Finalize() in W:ProjectsTestAppTestAppProgram.cs:line 11  
```
  
There are a couple of ways to deal with this - the best is to do all your testing and close the holes.  You really shouldn't have unhandled exceptions loose in your program.  But for a really large program, or one where you don't know where the exception is happening, Microsoft has given you a couple of options.  The first one is the **<legacyUnhandledExceptionPolicy> ** tag which you can put into your **app.config** file:

```xml
<configuration>
 <runtime>
 <legacyUnhandledExceptionPolicy enabled="1"/>  <runtime>  
<configuration>
```

This will essentially revert the CLR to the 1.1 behavior.  Consider it a short-term fix because it will ultimately mask issues with your application that are really bugs.

Along with the above, you can also be notified about unhandled exceptions prior to the AppDomain being unloaded.  This is done through several different methods based on the type of application. The first basic method is through the AppDomain.UnhandledException event which is raised for Console applications:

```csharp
static void Main()
{
    AppDomain.CurrentDomain.UnhandledException += delegate(object sender, UnhandledExceptionEventArgs e)
    {
        Console.WriteLine("{0} - IsTerminating = {1}", e.ExceptionObject.ToString(), e.IsTerminating);
    };

    for(; ;)
        new BadClass();
}
```

In this case, the application will still be terminated, _but_ we will now get notified right before termination.  This allows our application to log the error (using the nifty **System.Diagnostics.TraceSource** support or through the EventLog) so we at least know what happened.

For Windows Forms applications, things are a little different -- instead of the **AppDomain.UnhandledException** event being raised, the Windows Forms infrastructure will raise the **Application.ThreadException** event for exceptions that occur on the primary thread. The default behavior for this handler is to display the friendly **System.Windows.Forms.ThreadException** dialog:

![](/images/unhandled_exception.jpg)

If the exception occurs in the primary (message pumping) thread, the user will see the above dialog box and be given the choice to terminate the application or not.  The code behind it looks something like:

```csharp
void Application_ThreadException(object sender, ThreadExceptionEventArgs e)
{
    try
    {
        // Call user override
        if (this.threadExceptionHandler != null)
        {
            this.threadExceptionHandler(Thread.CurrentThread, new ThreadExceptionEventArgs(e.Exception));
        }
        else
        {
            using (ThreadExceptionDialog excptDlg = new ThreadExceptionDialog(e.Exception))  
            {
                DialogResult result = excptDlg.ShowDialog();
                if (result == DialogResult.Abort) Application.Exit();  
            }  
      }  
    }
    catch
    {  
    }  
}
```

That's great for the main thread, but if the exception occurs in a secondary thread, then the application will still be terminated after raising the **AppDomain.UnhandledException** event.  So, for non-UI threads, you must still register the unhandled exception handler on the AppDomain, log the failure and then watch your application die.  Having two handlers can be a pain and if you want to have the application terminate on _any_ unhandled exception, you can direct Windows Forms to _not_ catch primary thread exceptions automatically by using the **Application.SetUnhandledExceptionBehavior** method:

```csharp
[STAThread]  
static void Main()  
{  
    Application.SetUnhandledExceptionMode(UnhandledExceptionMode.ThrowException);  
    Application.Run(new Form1());  
}
```

The above line will cause exceptions thrown in the main thread to be unhandled - thereby triggering the **AppDomain.UnhandledException** event.

For ASP.NET applications, things are also a bit different.  First, be aware that any exception thrown during a Page request (the normal Page rendering process) will be handled automatically by ASP.NET and rendered back to the client based on the error settings in web.config.  If I throw an exception while rendering the page, then I'll get an HTML response like:

```output
Server Error in '/TestWebSite' Application.  
  
Description: An unhandled exception occurred during the execution of the current web request. Please review the stack trace for more information about the error and where it originated in the code.
```

The important thing here is that the AppDomain _continues to run_.  I can tell ASP.NET to redirect to a specific page through the `<customErrors>` section of the web.config.  Or, I can catch these exceptions and handle them myself through the global.asax **Application_Error** method (which is hooking the **HttpApplication.Error** event):

```csharp
void Application_Error(object sender, EventArgs e)  
{  
    Exception ex = Server.GetLastError();
    Context.ClearError();    // Stop default error reporting
    Response.Write("Application Error:");  
    Response.Write("****" + Server.HtmlEncode(ex.ToString()) + "****");
}
```

A more popular way to display errors is to cache off the last error in a session variable and then execute **Server.Transfer** to some error.aspx page:

```csharp
void Application_Error(object sender, EventArgs e)  
{  
    Exception ex = Server.GetLastError();
    Session["LastException"] = ex;
    Server.Transfer("myerrorpage.aspx");
}
```

Unfortunately, just like Windows Forms, if an unhandled exception occurs on a separate thread (such as a Threadpool thread), or outside the request processing framework, then the AppDomain is terminated.  This has significant consequences for ASP.NET applications - because the website itself is cycled.  This means you have lost session state and your users are going to notice the website outage.  Most of the time, there isn't much information to work off of -- you end up with a cryptic Event Log entry like:

```output
EventType clr20r3, P1 w3wp.exe, P2 6.0.3790.1830, P3 4333d6f1, P4 app_web_5hu0gabf, P5 0.0.0.0, P6 4415a8be, P7 7, P8 a, P9 system.exception, P10 NIL.
```

In order to get more information, we need to hook the **AppDomain.UnhandledException** event and then log the information, just like we have for Windows Forms and Console applications.  We could put a handler into our global.asax, but that would only handle that one web application.  It would be better to create a new HTTP Module with the event handler and hook that up through the web.config.  Luckily for me, Microsoft has already published a KB article which shows off this exact solution - [http://support.microsoft.com/?id=911816](http://support.microsoft.com/?id=911816).  In this article, Microsoft details creating a new HttpModule class which will log the exception information to the event log so you will know exactly where the exception occurred and can provide your own exception handling.  Unfortunately, without the above `<legacyUnhandledExceptionPolicy>` entry, the website will still be cycled.

Bottom line is: be aware of this new, breaking change -- look for places where you might not be handling exceptions properly and implement good audit trail mechanisms to log failures when/if they occur.  This will allow you to find and fix issues before they become production problems as you port your legacy code over to .NET 2.0!
