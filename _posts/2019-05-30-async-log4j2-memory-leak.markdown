---
layout: post
title:  "Async Log4j2, memory leak?"
author: frandorado
categories: [spring]
tags: [spring, log4j, log4j2, async, apache, logging, memory, leak]
image: assets/images/posts/2019-05-30/memory-leak.png
toc: true
---

Recently in a project that we are been working on, we found a significant increment in the boot time of a Spring Boot application. This happened when we activated asynchronous logs with Apache Log4j2.

## What
In order to improve the performance in our application, we decided to use Apache Log4j2 in asynchronous mode. In the majority of our applications we are using a very limited memory, and when we added this asynchronous feature we found that the boot time had incremented **from 30 seconds to more than 1 minute in some cases**. It was ridiculous!

Analyzing with `jvisualvm` and making a memory dump of our application in asynchronous mode we found that ~40Mb of memory is used by `RingBufferLogEvent`:

![Async log]({{site.url}}/assets/images/posts/2019-05-30/async-log-memory.png "Async log")

However, making a memory dump using synchronous mode we foud this:

![Sync log]({{site.url}}/assets/images/posts/2019-05-30/sync-memory-log.png "Sync log")

## Why
The implementation of Apache Log4j2 in async mode uses a RingBuffer to buffering all the logs content. By default uses 262144 slots (256 * 1024). This causes an initial memory reserve of approximately 40 megabytes and in a environment with a limited memory causes the memory head to be always full and therefore the starting slowdown.

## How
In the case that you don't need so many slots, we have two possibilities to reduce the memory:

* Use synchronous mode
* Use asynchronous mode with a shorter RingBuffer size. To do this we have to decrement the number of slots using the next property. The minimum value is 128. An example of 32768 (32 * 1024) slots that reserves about 5 Mb of memory.

Log4j >= 2.10
```
log4j2.asyncLoggerRingBufferSize=32768
```

Log4j < 2.10
```
AsyncLogger.RingBufferSize=32768
```

Here a table with approximately the reserved memory depending on the slots

```
| Slots     | Memory |
|-----------|--------|
| 128       | <1Kb   |
| 32*1024   | ~5 Mb  |
| 256*1024  | ~40 Mb |
```

More info in [Apache Log4j2](https://logging.apache.org/log4j/2.x/manual/async.html)