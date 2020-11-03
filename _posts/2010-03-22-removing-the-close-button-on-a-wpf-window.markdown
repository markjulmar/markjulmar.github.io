---
title: Removing the Close Button on a WPF window
date: 2010-03-22T00:00:00.0000000
layout: post
categories: wpf
tags: wpf dotnet
---

Today I was building a simple simulator to test some events to a new piece of hardware I’m working on.  Of course, I’m using WPF to show the simulator – and I wanted to create a topmost window that did not have a Close button on it.  Imagine my surprise when I realized there was not a set of flags you could supply to the Window object to actually achieve this result!

However, with a little Win32 mojo we can get the desired effect:

```csharp
public partial class MainWindow  
{  
    public MainWindow()  
    {  
        SourceInitialized += MainWindow_SourceInitialized;  
        InitializeComponent();  
    }

    void MainWindow_SourceInitialized(object sender, EventArgs e)  
    {  
        WindowInteropHelper wih = new WindowInteropHelper(this);  
        int style = GetWindowLong(wih.Handle, GWL_STYLE);  
        SetWindowLong(wih.Handle, GWL_STYLE, style & ~WS_SYSMENU);  
    }

    private const int GWL_STYLE = -16;  
    private const int WS_SYSMENU = 0x00080000;

    [DllImport("user32.dll")]  
    private extern static int SetWindowLong(IntPtr hwnd, int index, int value);  
    [DllImport("user32.dll")]  
    private extern static int GetWindowLong(IntPtr hwnd, int index);  
}
```

The key here is hooking the **SourceInitialized** event and then using the **SetWindowLong** function to strip off the **WS_SYSMENU** bit.  You can just cut/paste this code right into your solution.
