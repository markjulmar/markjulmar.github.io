---
title: Creating TFS Work Items programatically
date: 2006-08-28T00:00:00.0000000
layout: post
categories: dotnet
tags: dotnet
---

One question I get a lot when teaching the VSTS course is "How can I move tasks or bugs from our existing system into TFS?"  
  
The answer is quite simple: write a program that utilizes the Team Systems Object Model.  
  
The Object Model is documented partially in the VSIP SDK -- if you intend to work with it, I highly recommend you go and grab that SDK from Microsoft, but here is a simple example of creating a bug in the first Team Project available on the server named **TFSServer** (you can imagine the rest which uses ADO.NET to pull your existing tracking tickets from whatever system you have):  

```csharp  
using System;
using System.Collections;
using System.Collections.Generic;
using System.Text;
using Microsoft.TeamFoundation.Client;
using Microsoft.TeamFoundation.WorkItemTracking.Client;

namespace EnterWorkItems
{
    class Program
    {
        static void Main(string\[\] args)
        {
            TeamFoundationServer tfs = new TeamFoundationServer("TFSServer");
            WorkItemStore wis = (WorkItemStore) tfs.GetService(typeof(WorkItemStore));
            Project teamProject = wis.Projects[0];
            foreach (WorkItemType wit in teamProject.WorkItemTypes)
                Console.WriteLine(wit.Name);

            WorkItemCollection wic = wis.Query("Project='Test' AND Type='Bug'");
            foreach (WorkItem wiEntry in wic)
            {
                ...
            }

            WorkItemType witBug = teamProject.WorkItemTypes["Bug"];
            if (witBug != null)
            {
                Console.WriteLine("Adding new bug to Team Project {0}", teamProject.Name);
                WorkItem wi = new WorkItem(witBug);
                wi.Description = "This is a sample bug which was added through the object model";
                wi.Reason = "New";
                wi.Title = "You have Bugs! \[Ding\]";
                wi.Save();
                Console.WriteLine("Added Work Item # {0} created by {1}", wi.Id, wi.CreatedBy);
            }
        }
   }
}
```
  
Breaking this down a bit, the first main piece of code is the connection to the Team Server itself. This is accomplished on line 14 by creating a new **TeamFoundationServer** object. An optional constructor allows for credentials to be provided, otherwise it logs on as the current principle.  
  
Next, we retrieve the **WorkItemStore**. Almost everything in the object model is accessed through the **IServiceProvider** interface which provides a **GetService** method where you pass in the **System.Type** object you want to work with. This is a nice versioning technique that is utilized in many other Microsoft technologies as well.  
  
With the WorkItemStore, you can then query work item type definitions, one of which is necessary to create a **WorkItem**. You fill in the details for the Work Item you want to create and call the **Save** method to commit the changes to the TFS store. The WorkItem id will then be valid and could be added to the existing bug tracking system as a forward link to the new information if you wanted.
