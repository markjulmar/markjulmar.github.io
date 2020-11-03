---
title: Adding ILDasm to VS.NET with a keyboard shortcut
date: 2008-05-13T00:00:00.0000000
layout: post
categories: dotnet
tags: dotnet
---

I often show students in my classes how to add ILDasm to their tools menu - it's handy because you can then click on ILDasm and it will open the current module allowing you to look at the manifest, IL, etc.  
  
1. Select **Tools | External Tools**. You’ll get the tools dialog:

	![Tools1.png](/images/Tools1.png)  
  
1. Enter ILDASM for the title and set the Command to the path for ILDASM. The path varies a bit from version to version of Visual Studio. For VS2008 you will find it at **C:Program Files\\Microsoft SDKs\\Windows\\v6.0A\\bin\\ildasm.exe**  
  
1. Click the button to the right of Arguments and select "Target Path" and select the "Use Output Window" checkbox.  
  
	![](/images/Tools2.png)  
  
1. You should now be able to access Tools | ILDASM.  
  
One student in another class recently asked how to get a keyboard shortcut to this. [Michael Kennedy](http://www.michaelckennedy.net) - our resident VS.NET shortcut king showed exactly how to do this. It turns out to be really simple -- go to **Tools | Options** and select **Keyboard**. Then 1., 2., 3.:
  
![](/images/CustomLauncher.png)
