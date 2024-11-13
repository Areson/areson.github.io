---
title: "Throttling with Azure Functions"
categories:
  - Azure
tags:
  - Azure Functions
  - cloud
---

I've been struggling a lot lately to make a practical implementation using [Azure Functions](https://azure.microsoft.com/en-us/services/functions/) for high-volume processing when I need the resulting data to be persisted into a SQL database. It's not that it _can't_ do it...it's actually quite straightforward to do so. The issue arises when you start ramping up the throughput so that you can process the data quickly. The scaling built into Azure Functions can quickly overwhelm whatever it is pointing at. <!-- more -->

As it stands now the options for throttling Azure Functions, using the consumption model, are pretty limited. It's either a trickle or a fire hose. The result is that there is only so much one can do to fine-tune this, and you either need to scale up whatever it is persisting data to (more costs) or try to manage the incoming data (which may not be practical). If you are desperate you could feed data from Azure Function into some sort processing hosted on a VM or on premise, but then that defeats the point of going serverless _and_ adds extra complexity.

I've hit on a few solutions so help mitigate this that I wanted to outline. They aren't really true throttling...more like approaches for trying to minimize the liklihood of overwhelming everything. That being said, it's still possible to run into issues if you have enough scale out.