---
title: Moving from synchronous APIs to async APIs
date: 2013-07-25T00:00:00.0000000
layout: post
categories: dotnet
tags: dotnet
---

One of the main features added to .NET 4.5 is a whole new set of asynchronous APIs that mirror the current synchronous versions.  These were added primarily for Windows Store Apps (where they actually _replace_ the existing APIs) but also to improve the responsiveness of UI applications with the advent of the async/await keywords in C# 5.

The [.NET Bio project](https://github.com/dotnetbio/) has a whole set of APIs to hit common bioinformatic web services such as [BLAST](http://blast.ncbi.nlm.nih.gov/) which searches various databases trying to find matching sequences to new data.  When these APIs were designed and written, a polling mechanism was built to support the async nature of the web service calls.  So, a typical program to hit a BLAST service might look something like this:

```csharp
using System;
using System.Threading;
using Bio;
using Bio.Web;
using Bio.Web.Blast;

namespace BlastExample
{
    class Program
    {
        private static void Main(string[] args)
        {
            const string seq = "ACCTCCACTAGCTTTGTTTGTAGTGATGCTCTGTAGCACCACTGG" +
                               "GAAGCCCTTTAATGAATGTGCCTTTCCGCAAATCACACACACACAAATACACTTATAGAAACAAGGTGATTTTCTTGAAA" +
                               "TAATAAAACAAAATTTGGAAGAAGATTTTTACTGTCTTAGGAAAAGTAAGGCATTGGAAGGTGGCTAGGTATGACATATG" +
                               "AAGTTGCATTTTAAAACTGGAATTGGACAACTGATATTCAGTGATATTTATGCTACTACCTTCTAGAATCGAGAGCATGC" +
                               "ACCCCACTCTGTACTCTTGCCTGGAGAATCCATGATGAGAGCCTGGTAGGCTGCAGTCCATGGGGTCACACAGAGTCGGA" +
                               "CATGACTGAGCGACTTCACTTTCACTTTTCAATTTCATGCATTGGAGCCGGAAATGGCAACCCACTCCAGTGTTCTTGCC" +
                               "TGGAGAATCCCAGGGATGGGGAAGCCTGGTGGGCTGCTGTCTATGGGGTCGCAGAGAGTCAGACACGACTGAAGTGACTT" +
                               "AGCAGCAACCTTCTGGAATAAACGCCTCAGGCTTTAAACTCTGGCTTGACCATTCACTAGCCATGGGATCCACTAGAGTC" +
                               "GACCTGCAGGCATGCAAGC";

            ISequence sequence = new Sequence(Alphabets.DNA, seq);

            var blastService = new NCBIBlastHandler(new ConfigParameters
                {UseBrowserProxy = false, UseAsyncMode = true});

            // Set database and program to search.
            var searchParams = new BlastParameters();
            searchParams.Add("Program", "blastn");
            searchParams.Add("Database", "ecoli");

            string jobId;
            try
            {
                jobId = blastService.SubmitRequest(sequence, searchParams);
            }
            catch (Exception ex)
            {
                Console.WriteLine("Service is not available: " + ex.Message);
                return;
            }

            ServiceRequestInformation info = blastService.GetRequestStatus(jobId);
            if (info.Status != ServiceRequestStatus.Waiting
                && info.Status != ServiceRequestStatus.Ready)
            {
                Console.WriteLine("Service is not ready or waiting.");
                return;
            }

            for (int attempt = 0; attempt < 10; attempt++)
            {
                info = blastService.GetRequestStatus(jobId);
                if (info.Status == ServiceRequestStatus.Error || info.Status == ServiceRequestStatus.Canceled)
                {
                    Console.WriteLine("Requested failed, status is: {0} - {1}", info.Status, info.StatusInformation);
                    return;
                }

                if (info.Status != ServiceRequestStatus.Ready)
                {
                    Thread.Sleep(info.Status == ServiceRequestStatus.Waiting ||
                                 info.Status == ServiceRequestStatus.Queued
                        ? 5000 * attempt
                        : 1);
                }
                else break;
            }

            var results = blastService.FetchResultsSync(jobId, searchParams);
            if (results == null)
            {
                Console.WriteLine("No results returned.");
                return;
            }

            foreach (var r in results)
            {
                var md = r.Metadata;
                if (md != null)
                {
                    Console.WriteLine("Metadata:");
                    Console.WriteLine("Database: {0}, Program: {1}, Version: {2}", md.Database, md.Program, md.Version);
                    Console.WriteLine("Query Sequence: {0}", md.QuerySequence);
                    Console.WriteLine("Parameter Query: {0}", md.ParameterEntrezQuery);
                    Console.WriteLine("Parameter Expect: {0}", md.ParameterExpect);
                    Console.WriteLine("Parameter Filter: {0}", md.ParameterFilter);
                    Console.WriteLine("Gap Extend: {0}, Open: {1}", md.ParameterGapExtend, md.ParameterGapOpen);
                    Console.WriteLine("Include: {0}, MatchScore: {1}, Mismatch Score: {2}", md.ParameterInclude,
                        md.ParameterMatchScore, md.ParameterMismatchScore);
                    Console.WriteLine("Matrix: {0}, Pattern: {1}", md.ParameterMatrix, md.ParameterPattern);
                    Console.WriteLine("Def: {0}, Query Id: {1}, Query Len: {2}", md.QueryDefinition, md.QueryId,
                        md.QueryLength);
                }

                foreach (var rec in r.Records)
                {
                    Console.WriteLine();
                    Console.WriteLine("Hits:");
                    foreach (var h in rec.Hits)
                    {
                        Console.WriteLine("Id: {0}, Length: {1}", h.Id, h.Length);
                        Console.WriteLine("Accession: {0}, Def: {1}", h.Accession, h.Def);
                        foreach (var hs in h.Hsps)
                        {
                            Console.WriteLine(" Alignment length: {0}", hs.AlignmentLength);
                            Console.WriteLine(" Bit Score: {0}, Density: {1}, EValue: {2}, Gaps: {3}", hs.BitScore,
                                hs.Density, hs.EValue, hs.Gaps);
                            Console.WriteLine(" Hit Sequence: {0}rnStart: {1}, End: {2}, Frame: {3}", hs.HitSequence,
                                hs.HitStart, hs.HitEnd, hs.HitFrame);
                        }
                    }
                }
            }
        }
    }
}
```

Notice where the code spins asking the web service for the current status of the request. It's inefficient (but necessary given the way BLAST is exposed by NCBI here), but also tedious and easy to code incorrectly. I think a better approach would be to rely on the new async pattern used in .NET 4.5 where a Task is returned from the API and we can use the **await** keyword to wait for the API to complete. For timeout (the retryCount shown above) we can use `CancellationToken` support with timeouts. Here's what the updated code might look like:

```csharp
using System;
using System.Threading;
using Bio;
using Bio.Web.Blast;
using System.Collections.Generic;

namespace BlastExample
{
    internal class Program
    {
        private static void Main(string[] args)
        {
            const string seq =
                    "ACCTCCACTAGCTTTGTTTGTAGTGATGCTCTGTAGCACCACTGG ..."; // ... use sequence data from above ... ISequence sequence = new Sequence(Alphabets.DNA, seq);

            var blastService = new NCBIBlastHandler();

            IList results;

            try
            {
                results = blastService
                    .SubmitRequestAsync(
                            // Sequence to find sequence,
                            // Set parameters using fluid syntax (replace Add eventually?
                            new BlastParameters()
                                .AddEx("Program", "blastn")
                                .AddEx("Database", "ecoli"),
                            // Wait up to 10 seconds for the response and support cancelation
                            new CancellationTokenSource(TimeSpan.FromSeconds(10)).Token).Result; 
            }
            catch(AggregateException ex)
            {
                Console.WriteLine("Service is not available: " + ex.Flatten().InnerExceptions[0].Message);
                return;
            }

            if (results == null)
            {
                Console.WriteLine("No results returned.");
                return;
            }

            // Remainder of the code is identical..
            foreach (var r in results) { ... }
        }
    }
}
```

I think this code reads much easier, is easier to program, and best of all, when used with a GUI program (or at least not called from **Main**), you can use await on the **SubmitRequestAsync** call to process the results. Here, because async/await is not usable from your Main method I am just getting the **Result** property which blocks the main thread. Using async/await would also simplify the error processing since it pulls exceptions out of the `AggregateException` for you.

I tried this out by creating a set of extension methods on the `NCBIBlastHandler` class and the `BlastParameters` class. These wrap the current polling logic into a `Task` which can then be awaited. I also added some Fluid API support to the `BlastParameters` so it can be created inline:

```csharp
new BlastParameters()
    .AddEx("Program", "blastn")
    .AddEx("Database", "ecoli"),
```

These would presumably replace the existing `Add` methods already on the class - it's not a breaking change since it just changes the return type which for existing usages would be ignored anyway (as it's void today).

Here are the extension methods:

```csharp
using System;
using System.Collections.Generic;
using System.Runtime.Serialization;
using System.Threading;
using System.Threading.Tasks;
using Bio;
using Bio.Web;
using Bio.Web.Blast;

namespace BlastExample
{
    public static class NcbiBlastRequestExtensions
    {
        public static BlastParameters AddEx(this BlastParameters bp, string parameterName, string parameterValue)
        {
            bp.Add(parameterName, parameterValue);
            return bp;
        }

        public static Task<IList> SubmitRequestAsync(this NCBIBlastHandler handler, ISequence sequence,
            BlastParameters searchParameters)
        {
            return handler.SubmitRequestAsync(new List(new[] {sequence}), searchParameters, CancellationToken.None);
        }

        public static Task<IList> SubmitRequestAsync(this NCBIBlastHandler handler, ISequence sequence,
            BlastParameters searchParameters, CancellationToken cancellationToken)
        {
            return handler.SubmitRequestAsync(new List(new[] {sequence}), searchParameters, cancellationToken);
        }

        public static Task<IList> SubmitRequestAsync(this NCBIBlastHandler handler, IList sequences,
            BlastParameters searchParameters)
        {
            return handler.SubmitRequestAsync(sequences, searchParameters, CancellationToken.None);
        }

        public static Task<IList> SubmitRequestAsync(this NCBIBlastHandler handler, IList sequences,
            BlastParameters searchParameters, CancellationToken cancellationToken)
        {
            var jobId = handler.SubmitRequest(sequences, searchParameters);
            return Task.Run(async () =>
            {
                cancellationToken.ThrowIfCancellationRequested();

                ServiceRequestInformation info = handler.GetRequestStatus(jobId);
                if (info.Status != ServiceRequestStatus.Waiting && info.Status != ServiceRequestStatus.Ready &&
                    info.Status != ServiceRequestStatus.Queued)
                {
                    throw new BlastException() {ServiceRequestInformation = info};
                }

                do
                {
                    cancellationToken.ThrowIfCancellationRequested();

                    if (info.Status != ServiceRequestStatus.Ready)
                    {
                        if (info.Status == ServiceRequestStatus.Error || info.Status == ServiceRequestStatus.Canceled)
                        {
                            throw new BlastException() {ServiceRequestInformation = info};
                        }

                        await Task.Delay(1000, cancellationToken);
                    }

                    info = handler.GetRequestStatus(jobId);
                } while (info.Status != ServiceRequestStatus.Ready);

                cancellationToken.ThrowIfCancellationRequested();

                if (info.Status != ServiceRequestStatus.Ready)
                {
                    throw new BlastException("Timed out") {ServiceRequestInformation = info};
                }

                return handler.FetchResultsSync(jobId, searchParameters);
            }, cancellationToken);
        }
    }

    [Serializable]
    public class BlastException : Exception
    {
        public ServiceRequestInformation ServiceRequestInformation { get; set; }

        public BlastException()
        {
        }

        public BlastException(string message) : base(message)
        {
        }

        public BlastException(string message, Exception inner) : base(message, inner)
        {
        }

        protected BlastException(SerializationInfo info, StreamingContext context) : base(info, context)
        {
        }
    }
}
```

These are by no means production quality code, but they show off what could be done in the API. I like this approach and for GUI apps it makes total sense. It also allows for 100% backward compatibility which for some projects is crucial.

If you are interested in running the code, here's the final async example: [AsyncBlastExample.zip](/samples/Async.BlastExample.zip)
