---
title: Moving Work Items from one Team Project to another
date: 2006-07-17T00:00:00.0000000
layout: post
categories: dotnet
tags: dotnet
---

I was having lunch with an associate a while back and he mentioned a need to move a work item from one team project to another. While there isn't any direct support for this from the Team Explorer interface, I figured it couldn't be too hard to manipulate the underlying database and achieve the results - it's just SQL Server right? I hadn't actually _looked_ at the schema mind you, I'm an optimist.  
  
It wasn't quite as simple as I thought - it felt like the database had been designed by the security group and intentionally obscured for privacy. The TFS database is fairly generic (so that it can add arbitrary items into the schema for custom project types) and has been somewhat secured to protect the code itself - the stored procedures are encrypted and SQL Profiler doesn't give much information on what's happening.  
  
So, I spent a week or so looking at the schema for the Work Item system and a lot of trial and error. End result is that I got a simple application to move work items around. This application does a couple of other things as well - lists projects, active web services, etc. I was mostly playing with the schema and trying to figure things out. It's just a console application but it does the job.  
  
[Here's](/samples/tfscmd.zip) the code, **use at your own risk (a.k.a. if it screws up your system, I can't help you)**. The source for the simple test project is included so you can see what was done and incorporate it into your own code base if you like.  
  
The program is fairly easy to use - a binary is included with the release. Issuing the command with no parameters generates a simple "Help" page -- 

```output
C:WorkTfsCmd> **tfscmd**
TfsCmd [command] [/t tfsserver] [/u user] [/p password] [params]  
ListProjects - Lists the active projects on the TF server.  
TfsCmd ListProjects  
ListServices - Displays the web services exposed by the TF server.  
TfsCmd ListWebServices  
WQL - Execute a WorkItem Query.  
TfsCmd Wql [query]  
MoveWorkItem - Moves a work item from one Team Project to another  
TfsCmd MoveWorkItem [id] [ProjectName]
```
  
Executing a command will use your logged on user/password (domain credentials) unless you supply a **/u** and **/p** parameter. For example, running the **ListProjects** command generates something like:  

```output
C:WorkTfsCmd> **tfscmd ListProjects**
Project Name: Test  
Status: WellFormed  
Guid: 8c5b175b-b0ef-46bb-9a9e-dde05bb145ac  
Process Template: MSF for Agile Software Development - v4.0  
Defined Properties: MSPROJ  
```
  
Listing the web services just hits the underlying database to get the information:  

```output  
C:WorkTfsCmd> **tfscmd ListServices**
BuildStoreService = http://(local):8080/Build/v1.0/BuildStore.asmx  
BuildControllerService = http://(local):8080/Build/v1.0/BuildController.  
IBISEnablement = http://(local):8080/Build/v1.0/Integration.asmx  
LinkingProviderService = http://(local):8080/Build/v1.0/Integration.asmx  
IProjectMaintenance = http://(local):8080/Build/v1.0/Integration.asmx  
PublishTestResultsBuildService = http://(local):8080/Build/v1.0/PublishT tsBuildService.asmx  
```

Moving work items is fairly easy - just provide the work item id (the numeric identifier) and the new project name - this must match an existing project in your TFS system. The tool will attempt to match up the iteration to a value in the new project - if it cannot, it will default to the first iteration available.  

```output  
C:WorkTfsCmd> **tfscmd MoveWorkItem 1 "Mark's New Project"**  
Work Item #1: "Create Project Definition" is currently located in project "Test Project (Iteration 0)"  
Moving work item to "Mark's New Project (Iteration 0)"  
```

If you have the TFS client open, then you should shut it down and reopen it before modifying the moved work item - the client appears to cache bits of information and I had some issues with moved items if I didn't clear the cache by closing the work item window and reopening it.  
  
Have fun!
