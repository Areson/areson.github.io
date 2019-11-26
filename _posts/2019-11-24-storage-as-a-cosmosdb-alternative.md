---
title: "Azure Storage as an Alternative to Cosmos DB"
categories:
  - Azure
tags:
  - Cosmos DB
  - Azure Storage
---

In a [previous post]({% post_url 2019-11-19-moving-from-batch-to-event %}) I gave a high-level view of our method for collecting data points over the course of a day and later aggregating them. To avoid persisting a lot of temporary, semi-structured data to our SQL database as we had done with our batch processing, I opted instead to store the individual observations together in a document store, namely [Cosmos DB](https://azure.microsoft.com/en-us/services/cosmos-db/):
>My thinking was that we could build up a structured JSON document with the observations throughout the day, grouped by the observation id. Cosmos was a document store, so the high volume of operations wouldn't be an issue and it would be less expensive than having a larger SQL instance. Additionally the built-in TTL on documents meant that we could let the system do the cleanup after a certain duration which would keep the document store small and reduce our costs.

Using Cosmos DB did work as I had expected and the only major issue I ran into while using it was having not provisioned _enough_ throughput to handle the volume of reads and writes we were issuing during the peak processing times we had. As I had noted:
>The current model that Microsoft offers though is based on a fixed rate of _available_ throughput, so you pay regardless of whether or not you use it. That meant that our high-volume but bursty processing would end up costing us. On top of that even the default throttling settings on Azure Functions quickly overwhelmed our provisioned throughput for our Cosmos DB, which meant that we had to throttle the functions more or provision more throughput for Cosmos.

I briefly explored dynamically managing the provisioning to scale it up or down as needed based on processing, but it ended up seeming like more complications to manage than we needed. I instead opted to start looking for an alternative method of storing the information we accumulated over the course of the day. I did consider just pushing this all back into SQL since that is what we had done before, but I didn't really want to fall back to that as the data format just didn't lend itself well to that and it was more overhead to our smaller SQL instances than I wanted to throw at them.

[Azure Storage](https://azure.microsoft.com/en-us/services/storage/) seemed like an option though I wasn't sure if it was really ideal. Would storing all of the "documents" as literal documents on a disk, but just in the cloud, do what I needed? Generally speaking, the answer was "yes". Azure Storage takes a very different approach to document storage due to the way it is setup. [It doesn't have provisioned throughput](https://docs.microsoft.com/en-us/azure/storage/common/storage-scalability-targets) that you can adjust but instead has limits on ingress and egress bandwidth, as well as simultaneous requests. For large files the throughput limits might not make sense, but for for small documents like we were using we wouldn't come anywhere near the multi-gigabit limits. The number of simultaneous requests at the time of writing was around 20,000...well above the amount of documents we were concurrently processing. It also allows for configurable "TTL-like" behavior based on the last modification time of a document, so I could still have the "auto-cleanup" functionality that I was leveraging with Cosmos DB.

Using Azure Storage also opened the door to archiving the detailed documents longer-term in case we needed to reprocess anything or do a different type of analysis than what we are doing now. The price of storage is significantly less, and it also provides the option of moving documents into cold-storage and archival storage at specified intervals, which is excellent for reducing costs. Coupled with being able to set a deletion time-frame means management of the entire life cycle of our detailed data from within Azure, no coding required. It's also possible to pipe data from Azure Storage into various analytical solutions, which is an added bonus if you need to use that sort of thing.

There are some caveats. Unlike Cosmos DB, Azure Storage makes no latency guarantees, which means some requests could take a relatively "long" time to return, compared to the millisecond times guaranteed by Cosmos. In cases where requests are being retrieved for near-realtime interactions, something like Azure Storage doesn't make sense. In our case, higher latency wasn't really a concern and neither were retries. We instead needed to be able to have a high level of concurrent processing to handle the number of different data points that we were receiving, which Azure Storage could handle perfectly well.

Our final model ended up using Blob Storage and looking something like this:
- Receive a message with an observation from a queue that had sessions enabled
- Get the document associated with the ID for the dataset and the time period it was associated with
- Update the document with the new observation
- Persist the updated document

{% include mermaid.html %}
<div class="mermaid">
	sequenceDiagram;
	participant Queue
	participant Service
	participant Storage
	Queue->>Service: Receive Session Message
	Storage->>Service: Retrieve Existing Document
	Service->>Service: Update Document
	Service->>Storage: Persist Document	
</div>
<br/>

Having sessions enabled on the queue was required to ensure that we didn't have any race conditions accessing a document. By grouping all observations from a particular dataset into a session we could ensure that our services would only process one observation at at time per data set, meaning there would be no competition trying to access the associated document. We also used this same set of information to name each blob so that it was unique and easy to access based on the data in each message. If the blob existed, we would update it and persist the changed document. If it didn't exist we created a new document from the observation and persist it. Once the document was persisted we would issue a [scheduled message]({% post_url 2019-11-20-event-driven-batch-processing %}) that would later cause our services to retrieve the document and perform our aggregation on the data it contained. Finally, I configured the data management on the storage container to delete any blobs that hadn't been processed within _X_ days, keeping our storage clean of any chaff leftover due to unresolved failures and errors.