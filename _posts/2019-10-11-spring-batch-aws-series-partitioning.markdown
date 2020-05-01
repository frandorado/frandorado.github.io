---
layout: post
title:  "Spring Batch AWS Series (III): Remote Partitioning"
date:   2019-10-11 02:00:00 +0200
published: true
categories: [spring]
tags: [spring, batch, integration, aws, remote, partitioning, chunking, sqs]
---

In this post we are going to implement our second step using remote partitioning strategy. The goal of this step is to calculate the next probable prime given a number.

![Remote Partitioning](https://raw.githubusercontent.com/frandorado/frandorado.github.io/master/static/img/_posts/springbatchaws/remotepartitioning.png "Remote Partitioning")

## Table of contents

1. [Description](#section1)
2. [Master project](#section2)
3. [Slave project](#section3)
4. [Results](#section4)
5. [How to run](#section5)
6. [References](#section6)

## <a name="section1"></a>1. Description

Remote Partition is similar to remote chunking because receives messages from a SQS queue but the response is not sent to other SQS queue. The context information is stored in database instead of message. The process is as follow:

1. The master sends messages for each partition to SQS and store the information for the slave in the DB. Then start polling the DB for results.
2. The slave reads the message from SQS, reads the associated information in DB and processes the work.
3. The slave marks in DB as completed when finishes the work.
4. When all the partitions are completed, the master continue to the next step.

## <a name="section2"></a>2. Master project

The master project will store the partition info in the database and will send a message to the slaves by each partition. Meanwhile it will check in database if all the slaves have finished. We could configure a timeout for the case that some slave don't response.

**"Master => Slave" Integration Flow (Request outbounds)**

```java
@Bean
public IntegrationFlow step2RequestIntegrationFlow() {
    return IntegrationFlows.from(step2RequestMessageChannel())
                           .transform(stepExecutionRequestToJsonTransformer)
                           .handle(buildMessageHandler())
                           .get();
}

private MessageHandler buildMessageHandler() {
    SqsMessageHandler sqsMessageHandler = new SqsMessageHandler(amazonSQSAsync);
    sqsMessageHandler.setQueue(REQUEST_QUEUE_NAME);
        
    return sqsMessageHandler;
}
```

1. Read from `step2RequestMessageChannel` channel. In this channel the `MessageChannelPartitionHandler` will write the info about the partition.
2. The previous channel will receive a `StepExecutionRequest` object with all the necessary information for the slaves. In this step we'll transform this object in Json using our own tranform `StepExecutionRequestToJsonTransformer`.
3. Send to SQS. This handler will send the previous message to SQS using `SqsMessageHandler`
4. The master will check the database using the jobExplorer with the poll interval indicated. The max timeout for wait the slaves also can be specified. All this parameters can be setted in `MessageChannelPartitionHandler`.

```java
@Bean
public PartitionHandler buildPartitionHandler() {
    MessageChannelPartitionHandler partitionHandler = new MessageChannelPartitionHandler();
        
    partitionHandler.setGridSize(10);
    partitionHandler.setStepName(STEP_NAME);
    partitionHandler.setMessagingOperations(buildMessagingTemplate(step2RequestMessageChannel()));
    partitionHandler.setJobExplorer(jobExplorer);
    partitionHandler.setTimeout(30 * 60 * 1000); // 30 minutes
    partitionHandler.setPollInterval(10000); // 10 seconds
        
    return partitionHandler;
}
```

## <a name="section3"></a>3. Slave project

**"Slave => Master" Integration Flow**

```java
@Bean
public IntegrationFlow step2RequestIntegrationFlow() {
    return IntegrationFlows.from(buildSqsMessageDrivenChannelAdapter())
                           .transform(jsonToStepExecutionRequestTransformer)
                           .handle(buildStepExecutionRequestHandler())
                           .channel("nullChannel")
                           .get();
}

private StepExecutionRequestHandler buildStepExecutionRequestHandler() {
    StepExecutionRequestHandler stepExecutionRequestHandler = new StepExecutionRequestHandler();

    stepExecutionRequestHandler.setStepLocator(beanFactoryStepLocator);
    stepExecutionRequestHandler.setJobExplorer(jobExplorer);

    return stepExecutionRequestHandler;
}
```
1. Read from SQS messages using `SqsMessageDrivenChannelAdapter` and send the json to the transformer.
2. Transform the received json in `StepExecutionRequest` using `JsonToStepExecutionRequestTransformer`
3. The `StepExecutionRequestHandler` will get the step in database and execute the step for each partition.
4. With `nullChannel` will finish the integration flow.

In this case the slave will indicate in database when its work has been finished so it's not necessary to use a response queue.

## <a name="section4"></a>4. Results

This is a example logs for an execution with 10 partitions with random numbers.

```
The number received is 7829 and the next probable prime is 7841
The number received is 1615 and the next probable prime is 1619
The number received is 4846 and the next probable prime is 4861
The number received is 6523 and the next probable prime is 6529
The number received is 276 and the next probable prime is 277
The number received is 857 and the next probable prime is 859
The number received is 2066 and the next probable prime is 2069
The number received is 4364 and the next probable prime is 4373
The number received is 3493 and the next probable prime is 3499
The number received is 2535 and the next probable prime is 2539
```

## <a name="section5"></a>5. How to run

* Run the docker-compose.yml file
  * `docker-compose up`
  * `TMPDIR=/private$TMPDIR docker-compose up` (MAC users)

* Create the queues if don't exists

  ```
  aws sqs create-queue --endpoint http://localhost:4576 --queue-name step2-request.fifo --attributes '{"FifoQueue": "true", "ContentBasedDeduplication":"true"}'
  ```

* Run one or more slaves using the main class `com.frandorado.springbatchawsintegrationslave.SpringBatchAwsIntegrationSlaveApplication`

* Run the master using the main application `com.frandorado.springbatchawsintegrationmaster.SpringBatchAwsIntegrationMasterApplication`

## <a name="section6"></a>6. References

[1] [Link to the project in Github](https://github.com/frandorado/spring-projects/tree/master/spring-batch-aws-integration)

[2] [Spring Batch AWS Series (I): Introduction](https://frandorado.github.io/spring/2019/07/29/spring-batch-aws-series-introduction.html)

[3] [Spring Batch AWS Series (II): Remote Chunking](https://frandorado.github.io/spring/2019/09/19/spring-batch-aws-series-chunking.html)

[4] [Spring Batch Framework](https://github.com/spring-projects/spring-batch)
