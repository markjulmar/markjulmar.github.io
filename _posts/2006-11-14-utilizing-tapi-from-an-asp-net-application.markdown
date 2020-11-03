---
title: Utilizing TAPI from an ASP.NET application
date: 2006-11-14T00:00:00.0000000
layout: post
categories: tapi
tags: tapi
---

I've gotten several email requests about using ATAPI from ASP.NET -- people have had trouble getting it to function properly so I figured I'd post an example of how it works. First, I'd recommend using ATAPI and **not** ITAPI3 - only because the latter pulls in COM objects which always makes things _much_ more complex.

The key thing to remember is where to hook up your events. Don't hook events inside your ASP.NET page-derived classes - that will keep the pages alive and cause memory leaks. Instead, use static methods (as presented here) or hook the events in **global.asax** or some other global shared class.

I created a simple dialer in ASP.NET 2.0 through the following steps:

1. Create an ASP.NET website
1. Add a reference to ATAPI.DLL
1. Add a Global Application Class (global.asax)
1. Create an instance of the `TapiManager` class in your global application class
1. Utilize the `TapiManager` from your pages

As an example, here is my **global.asax** class - I chose to store the **TapiManager** class into the Application property bag, but you could just make it a global as well if you prefer that approach.

```csharp
void Application_Start(object sender, EventArgs e) 
{
    // Code that runs on application startup
    JulMar.Atapi.TapiManager tapiManager = new JulMar.Atapi.TapiManager("TestWebApp");
    if (!tapiManager.Initialize())
        System.Diagnostics.EventLog.WriteEntry("Application", "TapiManager failed to initialize");
    else
    {
        Application["tapi"] = tapiManager;
    }
}
    
void Application_End(object sender, EventArgs e) 
{
    //  Code that runs on application shutdown
    JulMar.Atapi.TapiManager tapiManager = (JulMar.Atapi.TapiManager)Application["tapi"];
    if (tapiManager != null)
        tapiManager.Shutdown();
}
```

Next, I added some markup to my **default.aspx** file to give me some server-side controls to manipulate TAPI with:

```html
<%@ Page Language="C#" AutoEventWireup="true"  CodeFile="Default.aspx.cs" Inherits="_Default" %>
<html xmlns="http://www.w3.org/1999/xhtml" >
<head runat="server">
    <title>Untitled Page</title>
</head>
<body>
    <form id="form1" runat="server">
    <div>
        <h1>Sample TAPI Dialer</h1>
        <asp:Label runat="server" Text="Select Line:" />&nbsp;&nbsp;
        <asp:DropDownList ID="lineList" runat="server" />
        <br />
        <asp:Label runat="server" Text="Number to dial:" />&nbsp;&nbsp;
        <asp:TextBox runat="server" Width="100" ID="number" />&nbsp;&nbsp;
        <asp:Button runat="server" ID="dial" OnClick="DialNumber" Text=" Dial " />
        <asp:Button runat="server" ID="refresh" Text=" Refresh " />
        <br />
        <br />
        <asp:ListBox runat="server" ID="events" Width="400" Height="300" EnableViewState="false" />
    </div>
    </form>
</body>
</html>
```

Finally, I added the code-behind to actually do the dialing. As part of this, I hook the appropriate events using static methods - not instance methods so I don't keep the page alive longer than a single request. I also store all my TAPI events in a global string collection so that when the page is refreshed, I can display the current set of events.

```csharp
using System;
using System.Configuration;
using System.Web;
using System.Web.Security;
using System.Web.UI;
using System.Web.UI.WebControls;
using System.Collections.Specialized;
using JulMar.Atapi;
  
public partial class _Default : System.Web.UI.Page 
{
    static StringCollection data = new StringCollection();
	  
    protected void Page_Load(object sender, EventArgs e)
    {
        TapiManager tapiManager = (TapiManager)Application["tapi"];
        if (!Page.IsPostBack)
        {
            if (tapiManager != null)
                lineList.DataSource = tapiManager.Lines;
        }
  
        events.DataSource = data;
        DataBind();
    }
  
    protected void DialNumber(object sender, EventArgs e)
    {
        TapiManager tapiManager = (TapiManager)Application["tapi"];
        string lineName = lineList.SelectedValue;
  
        TapiLine line = tapiManager.GetLineByName(lineName, true);
        if (line != null)
        {
            if (!line.IsOpen)
            {
                try
                {
                    line.Open(MediaModes.InteractiveVoice);
                }
                catch
                {
                    line.Open(MediaModes.DataModem);
                }
  
                line.NewCall += new EventHandler<NewCallEventArgs>(line_NewCall);
                line.CallInfoChanged += new EventHandler<CallInfoChangeEventArgs>(line_CallInfoChanged);
                line.CallStateChanged += new EventHandler<CallStateEventArgs>(line_CallStateChanged);
            }
  
            if (number.Text.Length > 0)
            {
                TapiCall call = line.MakeCall(number.Text);
                data.Add(string.Format("Created call: {0}", call));
            }
        }
    }
  
    static void line_NewCall(object sender, NewCallEventArgs e)
    {
        data.Add(string.Format("New call: {0}, {1}", e.Call, e.Privilege));
    }
  
    static void line_CallStateChanged(object sender, CallStateEventArgs e)
    {
        data.Add(string.Format("CallState: {0} is now {1}", e.Call.ToString(), e.CallState));
    }
  
    static void line_CallInfoChanged(object sender, CallInfoChangeEventArgs e)
    {
        data.Add(string.Format("CallInfo: {0} {1}", e.Call.ToString(), e.Change));
    }
}
```

This deploys and executes properly on Windows 2003 Server and IIS6. I did not try it under XP or IIS5, although I see no reason why it would not work as shown. Hopefully this will help someone out there trying to use TAPI from within a website!
