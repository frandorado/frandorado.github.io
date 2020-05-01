---
layout: post
title:  "Spring Data Mongo using Mongo Cluster with Docker"
author: frandorado
categories: [spring]
tags: [spring, mongodb, cluster, replicaset, docker, mongo, docker-compose]
image: assets/images/posts/2019-04-16/mongo-cluster-with-spring-data-mongo.png
toc: true
---

In this post we are going to talk about how to configure a MongoDB Cluster in local using docker-compose and how to use it with Spring Data Mongo.

## Configure the Replica Set

For our example, we are going to configure a Replica Set with 3 nodes as shown in the following `docker-compose.yml` file:

```yaml
version: '3'

services:
  mongo1:
    image: mongo:3.6
    command: mongod --replSet rs0 --port 27017
    ports: 
      - 27017:27017
    networks: 
      - my-mongo-cluster

  mongo2:
    image: mongo:3.6
    command: mongod --replSet rs0 --port 27018
    ports: 
      - 27018:27018
    networks: 
      - my-mongo-cluster

  mongo3:
    image: mongo:3.6
    command: mongod --replSet rs0 --port 27019
    ports:  
      - 27019:27019
    networks: 
      - my-mongo-cluster
    
networks: 
  my-mongo-cluster:
```
Now, run the `docker-compose` file.

```bash
> docker-compose up
```

Once your Mongo containers are running, the final step is to configure the Cluster.

```
> docker-compose exec mongo1 mongo --eval "rs.initiate({_id : 'rs0','members' : [{_id : 0, host : 'mongo1:27017'},{_id : 1, host : 'mongo2:27018'},{_id : 2, host : 'mongo3:27019'}]})"
```

## Connect to the cluster

Firsly, modify your `/etc/hosts` file adding:

```
127.0.0.1 mongo1
127.0.0.1 mongo2
127.0.0.1 mongo3
```

Now you can connect to the Cluster with the next url:

```
mongodb://mongo1:27017,mongo2:27018,mongo3:27019/?replicaSet=rs0
```

## Testing in a Spring Boot Application

You could see a complete example of use in [Github][github-link]


[github-link]: https://github.com/frandorado/spring-projects/tree/master/spring-data-mongo-with-cluster