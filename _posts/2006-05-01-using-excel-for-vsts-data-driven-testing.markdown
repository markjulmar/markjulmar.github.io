---
title: Using Excel for VSTS Data Driven Testing
date: 2006-05-01T00:00:00.0000000
layout: post
categories: dotnet
tags: dotnet
---

A colleague of mine, Kev Jones, has posted some information on using a detached SQL Server database for driving VSTS unit tests which works great if you need a full blown SQL implementation.  However, if all you want is a simple data feed, you can also use an Excel file, and as it turns out, it's pretty easy to do.

Let's start by stealing Kev's sample, hopefully he won't mind -- say I have a starship class with a **FirePhotonTorpedo** method:

```csharp
public class Enterprise  
{  
    private int torpedosLeft = 10;  
    public int FirePhotonTorpedo(int count)  
    {  
        if (torpedosLeft < count)  
            throw new ArgumentException("count");  
        torpedosLeft -= count;  

        // Instruct bridge officer to "Fire!"  
        return torpedosLeft;  
    }  
}
```

The equivalent unit test might look something like:

```csharp
[TestMethod]  
public void TestFirePhotonTorpedo()  
{  
    Enterprise target = new Enterprise();  
    int torpedoCount = 5;  
    int expected = 5;  

    int actual = target.FirePhotonTorpedo(torpedoCount);  
    Assert.AreEqual(expected, actual, "FirePhotonTorpedo did not return the expected value.");  
}
```

This code just tests a specific case -- I would also need to write other unit tests for edge cases and exceptional cases.  I can do this several different ways I could do this:

1. Write a unit test for each specific case passing each value and testing the expected result.  
1. Put all the tests into this one unit test function -- asserting each validation as we go.  
1. Pull the input and expected out of a database and run them through the function.

This last step, as Kev details can be done from a SQL database - either one reachable by all the people who might run the unit tests, or as his blog shows from a .MDF file you check in with the project and then locally attach prior to running the unit tests.

So, to make an Excel data-driven test, the first step is to create an Excel document with columns for each of our pieces of data.  My sample Excel document has the following columns:

| Column | Description |
|--------|-------------|
| **ID** | A unique number identifying the row so we can use the Random data driven test. |
| **torpedoCount** | The input for our unit test. |
| **expected** | The expected result coming out of the function. |
| **shouldFail** | Whether the function will throw an exception. |

I can then enter a single test on each row of the given worksheet.  Here's a screen shot --

![](/images/datadriven-excel1.JPG)

You can create multiple "tables" by adding additional sheets to the worksheet.  Each sheet can be named as appropriate; I'm naming this one "TorpedoData".

Now, I need to add the appropriate attribute to my test case to indicate that it should pull the data from a data source and run the method once per row found.  The key is that _any_ ADO.NET data source can be used.  Here I will specify my Excel file which is named **UnitTests.xls** and indicate that the table itself is a particular sheet within the Excel document "TorpedoData":

```csharp
[TestMethod]  
[DeploymentItem("Tests.xls")] // Copies the file to the deployment directory  
[DataSource("System.Data.OleDb", // The provider  
    "Provider=Microsoft.Jet.OLEDB.4.0;Data Source=**'Tests.xls'**;Persist Security Info=False;Extended Properties='Excel 8.0'",  
    "TorpedoData$",      // The table name, in this case, the sheet name with a '$' appended.  
    DataAccessMethod.Sequential)]
public void TestFirePhotonTorpedo()  
{  
}
```

I also need to update the test case itself to use the data -- we do that through the TestContext.DataRow property that gives us access to the current row of data from our data source:

```csharp
public void TestFirePhotonTorpedo()  
{
    Enterprise target = new Enterprise();
    int torpedoCount = Convert.ToInt32(TestContext.DataRow["torpedoCount"]);  
    int expected = Convert.ToInt32(TestContext.DataRow["expected"]);  
    bool shouldFail = (bool)TestContext.DataRow["shouldFail"];  
  
    TestContext.WriteLine("Running test with TorpedoCount={0}, ExpectedCount={1}, ShouldFail={2}",  
            torpedoCount, expected, shouldFail);  
  
    try  
    {  
        int actual = target.FireTorpedos(torpedoCount);  
        Assert.AreEqual(expected, actual, "FireTorpedos did not return the expected value.");  
    }  
    catch (Exception ex)  
    {  
        if (!shouldFail)  
            Assert.Fail(string.Format("FireTorpedo threw exception {0}", ex.Message));  
   }  
}
```

So, here we will grab the data from the current row, converting it as necessary to the appropriate types, output a test line just to prove that we executed the method more than once and then run the test.  The test will compare the result with the expected database result and output a failure if they aren't the same.  If an exception is thrown, then the **shouldFail** must be true or that will be considered a failure.

This approach allows me to run through different scenarios very easily and I can just store the Excel worksheet right with my unit tests - make sure it's deployed to the target deployment directory (either through a **DeploymentItem** attribute, or through the test run configuration).  The size of my table here is about 14K, compared to the equivalent .MDF file which is over 2M for the same data.  If I didn't want to hardcode the filenames and such, I can also use an **app.config** file for the unit test and put the information there - just as Kev details.

Cool stuff indeed.
