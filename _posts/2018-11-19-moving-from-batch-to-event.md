---
title: "Moving from Batch to Event"
categories:
  - Azure
tags:
  - cloud
  - Azure Function
  - Cosmos DB
  - Azure SQL
---

We ingest fair amount of data from external sources and I wanted to move our infrastructure from our older, batch-style processing on a schedule to an event driven model backed by a messaging system. In our old model we'd periodically pull data from external systems and dump it into staging table in SQL and later move it to it's final location with a batch job. The data was large sets of time series data and doing a sorted insert directly into the final table wasn't practical while fetching the external data.

In a lot of ways this worked very well. The staging tables could handle a lot of data, and doing the inserts later was relatively fast once we had an opportunity to work with the data in the staging tables. The downside is that it was also _extremely_ brittle. Issues during pulling batches of external data could derail huge chunks of updates and the only option was to restart the job. Adding new sources of data was also expensive as I had to write additional code to setup another jobs specifically for new sources.

What I really wanted was the ability to dump a structered set of data from any source and have it end up in the right spot. In the event that something didn't go right or the data was badly formatted I only wanted the issue to effect that single piece of data while everything else continued to hum along. That pushed us towards messaging queues, and in our case [Azure Service Bus](https://azure.microsoft.com/en-us/services/service-bus/), as I wanted some of the additional ordering and sequencing it provided. At a high level that meant that instead of working with large batches of data we'd handle each piece as its own individual unit and persist it to the queue. From there it would be processed by anything that subscribed to that type of data, one of which would be a service to persist it to our SQL database.

On paper this sounded _great_. In reality it was a lot harder. When you start splitting out large batches of data into individual pieces you end up with a _lot_ of stuff floating around. To make it more challenging we had added some additional requirements in order to help decrease the bloat we had in our SQL server. We were pulling time-series data and in our original system we stored each individual data point in a series of observations. Later we would run an ETL job to create a set of daily statistics about the observations, which was the data we ultimately needed. Storing all of the observations going back _X_ periods took up a lot of space and meant that we had to use a larger SQL instance than we might have otherwise needed. We wanted to try and cut out the intermediate data and just store the final results. I ended up with a first pass that looked something like this:

* _Throughout the day_
	* Fetch the data from the external source
	* Persist the data to the service bus
	* Service pulls the data and
		* persists it to a document store keyed by the observation id
		* persists a deferred message to indicate there was data for the observation id that needed to be summarized
* _End of Day_
	* Fetch the delayed message
	* Pull data from the document store and perform the summarization
	* Persist the summary data to SQL

The document store in this case was a [Cosmos DB](https://azure.microsoft.com/en-us/services/cosmos-db/). My thinking was that we could build up a structured JSON document with the observations throughout the day, grouped by the observation id. Cosmos was a document store, so the high volume of operations wouldn't be an issue and it would be less expensive than having a larger SQL instance. Additionally the built-in TTL on documents meant that we could let the system do the cleanup after a certain duration which would keep the document store small and reduce our costs.

Some of this proved true. Cosmos _was_ able to keep up with the throughput _provided_ that you scaled it up large enough to do so. The current model that Microsoft offers though is based on a fixed rate of _available_ throughput, so you pay regardless of whether or not you use it. That meant that our high-volume but bursty processing would end up costing us. On top of that even the default throttling settings on Azure Functions quickly overwhelmed our provisioned throughput for our Cosmos DB, which meant that we had to throttle the functions more or provision more throughput for Cosmos. Throttling down the Azure Function meant that on paper we wouldn't be able to process the data we needed to quickly enough. Ramping up the Cosmos DB got expensive fast, and trying to setup our own system to monitor usage and dynamically modify the provisioned resources ended up looking more complicated than I wanted to take on.

I also thought I was being creative by persisting a [scheduled message](https://docs.microsoft.com/en-us/azure/service-bus-messaging/message-sequencing) to the message queue. I wanted to avoid scanning the document store for documents at the end of each day and decided to stick with the event model by creating a message that indicated that a document existed that needed to be summarized and persisted. What I ended up getting was a ton of duplicate messages for each document in the Cosmos DB due to the way the message deduplication in the service bus worked. These didn't cause problems as our operations were idempotent but it added a lot of chaff to our queue and increased costs.

All of these issues came along with the "_standard_" assortment of problems that one would normally see: timeouts, failed external dependencies, bad data, and throttling. Since this was all new to me I figured hooking up [Application Insights](https://docs.microsoft.com/en-us/azure/azure-monitor/app/app-insights-overview) would be good so I could keep an eye on errors and dependency failures, and respond quickly to issues as they occured. As a bonus Azure Functions supports the [Live Metrics Stream](https://docs.microsoft.com/en-us/azure/azure-monitor/app/live-stream) which meant I could monitor it all in real time as I did load tests. 

I quickly learned another lesson: A decent volume of messages could generate an _enormous_ amount of logging data, most of it uninteresting as it told me that things had worked as I expected. The volume was so large that we ended up paying close to ten times as much for our Application Insights instance as we did for the Azure Function.

I ended up working through all of these issue to end up with a solution I feel pretty happy with. Over the next several posts I'll outline the various approaches I took to deal with each of these issues.