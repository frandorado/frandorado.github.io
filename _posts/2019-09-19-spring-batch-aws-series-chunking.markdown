---
layout: post
title:  "Spring Batch AWS Series (II): Remote Chunking"
author: frandorado
categories: [spring]
tags: [spring, batch, integration, aws, remote, partitioning, chunking, sqs]
image: assets/images/posts/2019-09-19/remotechunking.png
toc: true
---


In this post we are going to implement the first step of our Spring Batch AWS Series using Remote Chunking. This step will send a chunk of numbers and the slaves will log the numbers that are prime. 


## Description

In Remote Chunking the Step processing is split across multiple processes, in our case communicating with each other using AWS SQS. This pattern is useful when the Master is not a bottleneck

![Remote Chunking detailed]({{site.url}}/assets/images/posts/2019-09-19/remotechunking2.png "Remote Chunking detailed")

With Remote Chunking the data is read by the `master` and sent to the `slaves` using SQS for processing. Once the process finishes, the result of the slaves will be returned to the master.

* `Master` does all the I/O operations
* `Slave` doesn't need database access to get the information. This arrives through SQS messages.

## Common configuration

We need to configure next tools:

* **Localstack**: It's used for simulate AWS SQS queues in local.
* **Postgresql**: It's used for store Spring Batch metadata about jobs and steps.

The next docker-compose.yml contains all the necesary for our projects:

```yml
version: '3'
services:

  postgres:
    image: postgres:9.6.8-alpine
    restart: always
    ports:
      - 5432:5432
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: password
      POSTGRES_DB: db

  localstack:
    image: localstack/localstack
    ports:
      - "4567-4583:4567-4583"
    environment:
      - SERVICES=sqs
      - DATA_DIR=${DATA_DIR- }
      - PORT_WEB_UI=${PORT_WEB_UI- }
      - LAMBDA_EXECUTOR=${LAMBDA_EXECUTOR- }
      - DOCKER_HOST=unix:///var/run/docker.sock
    volumes:
      - "${TMPDIR:-/tmp/localstack}:/tmp/localstack"
      - "/var/run/docker.sock:/var/run/docker.sock"
```

## Master project

This section shows the most important aspect of our master application, how to comunicate the requests and responses with the slave.

**"Master => Slave" Integration Flow (Request outbounds)**

```java
@Bean
public IntegrationFlow step1RequestIntegrationFlow() {
    return IntegrationFlows
              .from(step1RequestMesssageChannel())
              .transform(chunkRequestToJsonTransformer)
              .handle(sqsMessageHandler())
              .get();
}
```

1. Read from `step1RequestMessageChannel` channel. We have to define a special writer (`ChunkMessageChannelItemWriter`) that will write to the `step1RequestMessageChannel`.
2. The previous channel will receive a `ChunkRequest` object with all the necessary information for the slaves. In this step we'll transform this object in Json using our own tranform `ChunkRequestToJsonTransformer`.
3. Send to SQS. This handler will send the previous message to SQS using `SqsMessageHandler`

**"Slave => Master" Integration Flow (Response inbounds)**

```java
@Bean
public IntegrationFlow step1ResponseIntegrationFlow() {
    return IntegrationFlows
            .from(sqsMessageDrivenChannelAdapter())
            .transform(jsonToChunkResponseTransformer)
            .channel(step1ResponseMessageChannel()).get();
}
```
1. Read from SQS messages using `SqsMessageDrivenChannelAdapter` and send the json to the transformer.
2. Transform the received json in `ChunkResponse` using `JsonToChunkResponseTransformer`
3. Send the `ChunkResponse` to `step1ResponseMessageChannel` that is configured in `ChunkMessageChannelItemWriter` as reply channel. Now Spring will mark the step with the result received.

## Slave project

This section shows the communication with the master.

**"Master => Slave" Integration Flow (Request inbounds)**

```java
@Bean
public IntegrationFlow step1RequestIntegrationFlow() {
    return IntegrationFlows
              .from(buildRequestSqsMessageDrivenChannelAdapter())
              .transform(jsonToChunkRequestTransformer)
              .handle(step1ChunkProcessorChunkHandler())
              .channel(step1ResponseMessageChannel())
              .get();
}
```
1. Read from SQS messages using `SqsMessageDrivenChannelAdapter` and send the json to the transformer.
2. Transform the received json in `ChunkRequest` using `JsonToChunkRequestTranformer`
3. Send the `ChunkRequest` to `step1ChunkProcessorChunkHandler()` that is a special handler where you could define your processor and/or writter.
4. The result will be sent to `step1ResponseMessageChannel`

**"Slave => Master" Integration Flow (Response outbounds)**

```java
@Bean
public IntegrationFlow step1ResponseIntegrationFlow() {
    return IntegrationFlows
              .from(step1ResponseMessageChannel())
              .transform(chunkResponseToJsonTransformer)
              .handle(sqsMessageHandler())
              .get();
}
```
1. Read from `step1ResponseMessageChannel` and send the received `ChunkResponse` to the transformer.
2. Transform the object in json using `ChunkResponseToJsonTransformer`
3. Send the previous message to response queue in SQS.

## How to run

* Run the docker-compose.yml file
  * `docker-compose up`
  * `TMPDIR=/private$TMPDIR docker-compose up` (MAC users)

* Create the queues if don't exists

  ```
  aws sqs create-queue --endpoint http://localhost:4576 --queue-name step1-request.fifo --attributes '{"FifoQueue": "true", "ContentBasedDeduplication":"true"}'

  aws sqs create-queue --endpoint http://localhost:4576 --queue-name step1-response.fifo --attributes '{"FifoQueue": "true", "ContentBasedDeduplication":"true"}'

  ```

* Run one or more slaves using the main class `com.frandorado.springbatchawsintegrationslave.SpringBatchAwsIntegrationSlaveApplication`

* Run the master using the main application `com.frandorado.springbatchawsintegrationmaster.SpringBatchAwsIntegrationMasterApplication`

## References

[1] [Link to the project in Github](https://github.com/frandorado/spring-projects/tree/master/spring-batch-aws-integration)

[2] [Spring Batch AWS Series (I): Introduction]({{site.url}}/spring/2019/07/29/spring-batch-aws-series-introduction.html)

[3] [Spring Batch Framework](https://github.com/spring-projects/spring-batch)


