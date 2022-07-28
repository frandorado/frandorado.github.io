---
layout: post
title:  Spring Boot Starter Data Mongo with multiple databases
author: frandorado
categories: [spring]
tags: [spring, boot, starter, data, mongo, multiple, database]
image: assets/images/posts/2020-12-14/header.jpg
toc: false
hidden: false
---

In this post I am going to talk about how to configure multiple databases using Spring Boot Starter Data Mongo which is configured by default to support just one database.


## Disable Autoconfiguration

The first step is to disable the configuration for `MongoAutoConfiguration` and `MongoDataAutoconfiguration`. This could be made with two different ways:

One way is adding next property in your `application.properties` file.

```
spring.autoconfigure.exclude= \
  org.springframework.boot.autoconfigure.mongo.MongoAutoConfiguration,\
  org.springframework.boot.autoconfigure.data.mongo.MongoDataAutoConfiguration
```

The other one is excluding these classes directly in your SpringApplication annotation:

```java
@SpringBootApplication(exclude = {MongoAutoConfiguration.class, MongoDataAutoConfiguration.class})
```

## Add your database configuration

Now, it's time to add different database access. In this example, we will create a only mongo client with two different MongoTemplate objects, one per each database to create.

```java
@Configuration
public class MongoConf {
    
    @Bean
    public MongoClient mongoClient() {
        return MongoClients.create("mongodb://localhost:27017");
    }
    
    @Bean
    public MongoTemplate carMongoTemplate() {
        return new MongoTemplate(mongoClient(), "car");
    }
    
    @Bean
    public MongoTemplate truckMongoTemplate() {
        return new MongoTemplate(mongoClient(), "truck");
    }
    
}
```

Now, you could inject the `MongoTemplate` that you need in your code.

```java
@Autowired
MongoTemplate carMongoTemplate;

@Autowired
MongoTemplate truckMongoTemplate;
```



