---
title: "Event Driven Batch Processing"
categories:
  - Azure
tags:
  - Azure Service Bus
  - Scheduled Messages
---

{% include mermaid.html %}

One of the challenges I ran into trying to move to an event-driven model for processing our data what that some data streams still needed to be aggregated at some point. I mentioned in ["Moving from Batch to Event"]({% post_url 2018-11-19-moving-from-batch-to-event %}) that some of the data points we collect need to have a daily summary created and persisted to our SQL data warehouse. We could persist all the details to a staging table and run a daily ETL to do the aggregation, or maybe move it into the cloud and hook it up to a scheduled Azure Function that would run and do the same thing.

All of these batch-oriented approaches just feel too...well, batch-like. The move into an event-driven model meant that we were now creating messages for each data point collected throughout the day, and work only happened if it _needed_ to. We had a JSON document created for each data set that our services collected data for. Relying on a scheduled job instead of an event had some implications that I didn't like. The job might miss some documents due to timing or have errors while processing some of the documents. If the errors are transient then we'd have to wait until the _next_ run of the job to get that data persisted. We either end up waiting a long time to have data show up or we make the job run more frequently, which means we might spend a bunch of time running the job when there is nothing to do.

I wanted to stick with some sort of event-based processing still, to generate a message that indicated that processing needed to be done on a JSON document. The service that consumed that message would perform the processing and persist the result. The question was how to set something like that up.

Azure Service Bus supports [scheduled messages](https://docs.microsoft.com/en-us/azure/service-bus-messaging/message-sequencing) which allows you to publish a message and specify when it will be available for processing. I figured that each time an individual data point was added to a JSON document I could issue a scheduled message that indicated that there was processing that needed to be done. The message would be received later, e.g. the next day, and my final data would show up in the data warehouse. If a transient issue occurred while processing one of these messages only a single set of related data would be affected and we could just retry that message again after a delay.

One consequence of this approach is that it generated _a lot_ of messages. If I had 10 observations for a particular data set over the course of the day then I'd issue 10 scheduled messages to kick off the processing of those points. This seems like overkill and could potentially lead to some unintended consequences. In our case we use [message sessions](https://docs.microsoft.com/en-us/azure/service-bus-messaging/message-sessions) to prevent duplicates of the same message from being processed multiple times simultaneously by setting the session id for the processing messages to be `DataSetId-Date`. While that avoids odd race conditions I'll still have a lot of needless function invocations happening as only the first message out of all 10 is going to do anything.

I had hoped that enabling [duplicate detection](https://docs.microsoft.com/en-us/azure/service-bus-messaging/duplicate-detection) on the queue would resolve the issue for me. I didn't care about generating extra messages as long as only one survived to be processed. I tried enabling duplicate detection for a short lookback period on my queue, figuring that if I scheduled my messages to arrive at the same time that they'd all show up in that window and the duplicate detection would take care of everything. Turns out there is more going on under the hood with scheduled messages that prevents this from working the way that I'd like.

#### Deduplication: _This is how I thought it worked_
<div class="mermaid">
	sequenceDiagram;
	participant ServiceA
	participant Queue
	participant ServiceB
	ServiceA->>Queue: Scheduled Message
	Note over ServiceA: (delay)
	ServiceA->>Queue: Scheduled Message
	Note over ServiceA: (delay)
	ServiceA->>Queue: Scheduled Message
	Note over Queue: Scheduled Messages<br>Delivered
	loop Deduplication
		Queue->>Queue: 
	end
	Queue->>ServiceB: Message is Processed
</div>
<br/>

Scheduled messages actually end up being _two_ messages. The first is the message that we send to a queue that we want to be processed later. That message hits that queue and is processed immediately. This generates a second message, which is the one that gets sent later. This second message isn't subject to the same processing that the original message was and it isn't affected by deduplication. Only the _original_ message is. This means that if the original messages don't show up in the queue within my deduplication window, then they will all end up getting processed.

#### Deduplication: _This is how it actually works_
<div class="mermaid">
	sequenceDiagram;
	participant ServiceA
	participant Queue
	participant ServiceB
	ServiceA->>Queue: Scheduled Message	
	Note over ServiceA: (delay)
	ServiceA->>Queue: Scheduled Message
	Queue->>Queue: Deduplication Check
	Note over ServiceA: (delay)
	ServiceA->>Queue: Scheduled Message
	Queue->>Queue: Deduplication Check
	Note over Queue: Scheduled Messages<br>Delivered
	Queue->>ServiceB: Messages are Processed
</div>
<br/>

I didn't want to give up quite yet, so I turned to another feature available on Azure Service Bus: [autoforwarding](https://docs.microsoft.com/en-us/azure/service-bus-messaging/service-bus-auto-forwarding). Azure Service Bus lets you setup chaining between queues and have one queue forward the messages it receives to another. For handling my processing messages I wanted to have a long deduplication window to make sure that I only ended up handling one processing message per data set, per day, so 24 hours. This windows was much larger than what I wanted my normal queue to have so I created a second queue specifically to receive these processing messages. This "processing queue" had a 24 hour deduplication window and forwarded all messages to the normal queue.

Scheduled messages are handled by the queue that initially receives them, which was my processing queue. That queue had the 24 hour deduplication window, so all of my processing messages that came in over the day were deduplicated and only the original message was kept. When the scheduled delivery time arrived, the one remaining messages was materialized in the processing queue, which then forwarded it to the normal queue to be handled by the service. Now I have event-driven "batch" processing!

#### _The Final Flow_
<div class="mermaid">
	sequenceDiagram;
	participant ServiceA
	participant ProcessingQueue
	participant Queue
	participant ServiceB
	ServiceA->>ProcessingQueue: Scheduled Message
	Note over ServiceA: (delay)
	ServiceA->>ProcessingQueue: Scheduled Message
	ProcessingQueue->>ProcessingQueue: Deduplication
	Note over ServiceA: (delay)
	ServiceA->>ProcessingQueue: Scheduled Message
	ProcessingQueue->>ProcessingQueue: Deduplication
	Note over ProcessingQueue: Scheduled Message<br>Delivered
	ProcessingQueue->>Queue: Message Forwarded
	Queue->>ServiceB: Message is Processed
</div>
<br/>

There are some important considerations to this though. First, you need to make sure that you are setting you message IDs properly. The deduplication will use those, so you need to include any values you want to have considered for deduplication. As I noted above I used `DataSetId-Date` to make sure I would deduplicate per data set, per day. The second thing you need to keep in mind is that you'll need a separate queue for each period you want to deduplicate messages over. Want another batch of messages deduplicated over a 2 hour window? You'll need to setup a separate queue with a 2 hour deduplication window and forward it properly.

It's worth pointing out that you _could_ just take a single queue and put a really big deduplication window on it and manage your message IDs appropriately. For 24 hour messages you might have an ID like `DataSetId-Date` while a 2 hour message might be `DataSetId-TwoHourBucketTimestamp`. This would certainly work, but you'll have to watch your IDs careful. In our case the queue that our services are using handles a large variety of messages that need to be persisted, and we wanted a minimal deduplication window to handle cases where the same message got sent twice in a short period due to errors or outages. Having a longer window didn't make sense for us, and it can also contribute to more overhead. Per Microsoft's documentation: 
> Enabling duplicate detection and the size of the window directly impact the queue (and topic) throughput, since all recorded message-ids must be matched against the newly submitted message identifier.