---
title: .NET Bio is now available via NuGet!
date: 2012-02-24T00:00:00.0000000
layout: post
categories: bio
tags: bio
---

For those of you who have always wanted to write a cool bioinformatics analysis program using .NET I have some great news!  It’s now even easier to add support for the open-source .NET Bio project ([https://github.com/dotnetbio/bio](https://github.com/dotnetbio/bio)).

If you aren’t familiar with NuGet, it’s a package repository which allows you to add dependencies to a Visual Studio project directly from an online source (vs. just what you have on your machine) and keep them up to date as they change.  Check out [http://www.nuget.org](http://www.nuget.org) for more details on that, but let me quickly run down the steps on using it to grab .NET Bio 1.0.

First, make sure you have the NuGet package manager installed – that’s available from **Tools | Extension Manager**

![image](/images/image_thumb2.png "image")

Then scroll through the list to see if it’s present, or add it from the Online section in the dialog (search for NuGet):

![image](/images/image_thumb3.png "image")

Once that’s installed, you can access the NuGet repository from the **References** folder in your projects.  Just right-click like you normally would to add a reference, but select **Manage NuGet Packages** from the list.

![image](/images/image_thumb4.png "image")

Then, in the dialog, type **biology** or **bioinformatics** or some other bio-related keyword into the search field (.NET Bio might night be exact enough because so many projects include “.NET” in their name and it’s sorted by usage).  You should immediately find the new .NET Bio 1.0 package!

![image](/images/image_thumb5.png "image")

Just click **Install** to get it into your project – it will download, prompt you to accept the Apache license and then add the .DLLs and some sample code into your project.  It’s as easy as that!

![image](/images/image_thumb6.png "image")

The DLLs that are added depend on the target framework you are using.  There are two versions of .NET you can use:

**.NET Framework 4.0 Full –** this is the full framework where you expect all your clients to have a full install of .NET on their machine to run your program.  All features of .NET are available.

**.NET Framework 4.0 Client Profile –** this is a subset of the full framework that targets desktop applications – it is missing server-side support, ASP.NET, some WCF capabilities, etc.  This is the default target when you create a project using the Visual Studio project templates for WPF, Windows Forms or Console-based applications.

If you are using the client profile, you will get the core **Bio.dll** and the alignment / assembly algorithm DLLs (**Bio.PamSam, Bio.Padena, Bio.Comparative**).  If you target the full framework, Nuget will also install **Bio.WebServiceHandlers.dll** which enables support for online BLAST queries and such.  This DLL requires the full framework as it has a dependency on **System.Web** (which must also be added to your project).

In addition to the DLLs, you will notice there are some sample files added to the project in the **Samples** folder – since NuGet does not allow you to target a specific language, we chose to include both VB.NET and C# in the same package (we could have separate the packages and had a .NET Bio for C# and another for VB.NET but chose not to go that route at this point).  You can delete the files which do not apply to your project as they define exactly the same class (**BioUtilities**) which looks like this:

![image](/images/image_thumb7.png "image")

There are a hand-full of static methods on the type which support common usage scenarios to:

- Read a set of sequences from a FASTA file (**ParseFastA**)
- Write a set of sequences to a FASTA file (**ExportFastA**)
- Align two sequences against each other (**AlignSequences**)
- Align multiple sequences (2+) against each other (**DoMultipleSequenceAlignment**)
- Perform an assembly using Denovo (**DoDenovoAssembly**)
- Merge two sequence ranges (**DoBEDMerge**)

In addition, there are two other methods which are in the secondary source file(s): **BioUtilities.Blast.cs/vb** which are commented out by default to support online BLAST queries.  These require the full .NET framework and the **Bio.WebServiceHandlers.dll** reference.

- **SubmitBlastQuery** to submit a BLAST query to the NCBI service.
- **GetBlastResults** to retrieve the asynchronous results from the NCBI BLAST service.

The above is by no means complete coverage of .NET Bio, but provides some sample code to help you through common tasks.  The sharp-eyed might have noticed that the functions correspond exactly to the .NET Bio starter project you can use when you install the full .NET Bio install.

This is a great addition to NuGet and provides an easy way to begin to add bioinformatic analysis to your .NET applications quickly and easily.
