﻿# Stale indexes

RavenDB performs data indexing in a background thread, which is executed whenever new data comes in or existing data is updated. Running this as a background thread allows the server to respond quickly even when large amounts of data have changed, however in that case you may query stale indexes.

The notion of stale indexes comes from an observation deep in RavenDB's design, assuming that the user should never suffer from assigning the server big tasks. As far as RavenDB is concerned, it is better to be stale than offline, and as such it will return results to queries even if it knows they may not be as up-to-date as possible.

And indeed, RavenDB returns quickly for every client request, even if involves re-indexing hundreds of thousands of documents. And since the previous request has returned so quickly, the next query can be made a millisecond after that and results will be returned, but they will be marked as Stale.

## Checking for stale results

As part of the response when an index is queried, a property is attached indicating whether the results are stale - that is, whether there are currently any tasks outstanding against that index. It can be fetched using the `RavenQueryStatistics` object:

{CODE stale1@ClientApi\Querying\StaleIndexes.cs /}

When `IsStale` is true, that means someone probably added or changed a Product, and the indexes haven't had time to fully update before we queried.

For most cases you don't really care about that, however there are scenarios where you cannot work with data that could be stale.

## Explicitly waiting for non-stale results

When it is a requirement to get non-stale results back from a query, it is possible to specify this while querying:

{CODE stale2@ClientApi\Querying\StaleIndexes.cs /}

Note that in the sample above a time-out of 5 seconds was specified. While you can ask RavenDB to wait indefinitely until there are non-stale results, this should only be used in unit-testing, and never in a real-world application, unless you are 100% sure you understand the implications, and that is what you want to have.

## Setting cut-off point

Even when using `WaitForNonStaleResults` with a time-out like shown above, it is still possible to get back stale results - for example when a very lengthy indexing task is executing. A better approach to make sure you are working with non-stale results is to use a cut-off point and tell the server to use that as a base:

{CODE stale3@ClientApi\Querying\StaleIndexes.cs /}

This will make sure that you get the latest results up to that point in time. All pending tasks for changes occurred after this cut-off point will not be considered. And like before, a time-out can be set on that as well.

`WaitForNonStaleResultsAsOfNow` is also available, which is equivalent of calling `WaitForNonStaleResultsAsOf(DateTime.Now)`.

Another option is to use `WaitForNonStaleResultsAsOfLastWrite`, which does exactly what it says it do. It tracks the last write by the application, and uses that as the cutoff point. You might find it useful if you are working on the machines where clock synchronization might be an issue, since `WaitForNonStaleResultsAsOfLastWrite` doesn't use the machine time, but etag values for the writes. The time-out can apply as well.

The default time-out value is 15 seconds.

You can also setup the document store to always wait for the last write, like so:

{CODE stale4@ClientApi\Querying\StaleIndexes.cs /}

All queries in the store would behave as if `WaitForNonStaleResultsAsOfLastWrite` was applied to them.

{DANGER:Beware of `QueryYourWrites` overuse}
The indexing mechanism in RavenDB is built on a BASE model (Basically Available, Soft state, Eventual consistency). In order to avoid querying consistency pitfalls in the future you need to consider this at the data modeling phase.

The usage of `WaitForNonStaleResultsAsOfLastWrite` is usually reasonable on only very rare occassions. The need of taking advantage of `QueryYourWrites` convention is very often a symptom of deeper issues in an application model and misunderstanding of querying concepts in RavenDB. 
{DANGER/}