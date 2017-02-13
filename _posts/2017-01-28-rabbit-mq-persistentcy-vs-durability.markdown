---
layout: post
title: "RabbitMQ - Persistency vs Durability"
date: 2017-01-27T17:48:05+02:00
comments: true
tags: [rabbit-mq]
---

Once you start getting serious with RabbitMQ, around the time you graduate from running small tests on your laptop to actually processing real production data, is usually the time
you start to wonder about reliability; How will my server behave once it fails and restarts? And what will happen to my messages? You may have met the terms *durability* and *persistency*,
which sound like they have somewhat similar meaning. Why do we need both? and what are the differences between them?
<!-- more -->
Let's start by understanding what we're trying to achieve. Durability and persistence are needed when our messages are not expendable. Take for example 
a task queue scenario, where we push tasks to queue and workers pull tasks and execute them. We'd like to ensure that even if a broker fails and restarts 
while messages are still queued in it, these messages will not disappear. Otherwise we may face tasks which will never start.
Notice that this is not the default behavior: By default, once a RabbitMQ broker dies / restarts, all messages in it are gone.
To handle these scenario correctly, we need to get acquainted with these two terms, persistency and durability.

In short, to ensure that messages do survive server restarts, the message needs to:

1. Be declared as *persistent* message,
2. Be published into a *durable* exchange,
3. Be queued into a *durable* queue

So, why three steps? and what is the difference between durable and persistent?

*Durability* is a property of AMQP entities; queues and exchanges. Durable entities can survive server restarts, by being automatically recreated once the 
server gets back up. Notice that this does not imply that messages residing in this queue gets recreated in the newly created queue. For this to be the case,
the messages have to be *persistent*.

*Persistence* is a property of messages. To mark a message as persistent, set its delivery mode to 2. AMQP brokers handles persistent messages by writing them to disk 
inside a special persistency log file, allowing them to be restored in the event of server restart. Notice that persistent message has no effect inside a non-durable queue; When 
the server restarts, the queue will not be recreated and thus the messages will not be recreated from the persistency log file. This is also true for every non-durable queue that 
the message may be routed to: Once a message is routed to a non-durable queue it gets deleted from the persistency log file and is not considered persistent anymore.
The persistency log file gets cleaned periodically by garbage collection. Persistent messages are marked for garbage collection once they are consumed (and acknowledged) 
from a durable queue.

Message persistency, like everything else in life, comes at a price. In this case, the price is performance. Disk I/O is relatively slow, and writing every message to disk 
can slow the broker down and significantly reduce the number of messages it can process per second (Though SSD can help a lot here).



