---
layout: post
title:  "Spring Batch AWS Series (I): Introduction"
author: frandorado
categories: [spring]
tags: [spring, batch, integration, aws, remote, partitioning, chunking]
image: assets/images/posts/2019-07-29/steps.png
toc: true
---

This entry is the first in a series of posts about how to process different steps in Spring Batch **remotely**. In this project we are going to use Amazon SQS as communication channel between the remote processes.

## Introduction 

Spring Batch is a processing framework designed for robust execution of jobs. Each job is divided in steps and each step could be executed in a remote way using slaves. There are differents strategies to do this:

* **Remote Chunking:** This strategy is useful when we don't have bottleneck in reading or writing. [Spring Batch AWS Series (II): Remote Chunking]({{site.url}}/spring/2019/09/19/spring-batch-aws-series-chunking.html)
* **Remote Partitioning:** This strategy is useful when the bottleneck is in reading or writing. [Spring Batch AWS Series (III): Remote Partitioning]({{site.url}}/spring/2019/10/11/spring-batch-aws-series-partitioning.html)

## The project

The goal of this project is create a Job with diferent steps and each one will implement a different remote strategy.

* **Step1**: Using Remote Chunking
* **Step2**: Using Partitioning with pooling

We are going to divide in two projects:

* The `master` project that will be in charge of orchestrating the Job and sending the tasks to the slaves.
* The `slaves` will do the work and will report the result to master.

## References

[1] Spring Batch Framework [Github][github-link]


[github-link]: https://github.com/spring-projects/spring-batch

