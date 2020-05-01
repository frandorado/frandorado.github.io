---
layout: post
title:  "Example of MQTT with Mosquitto"
author: frandorado
categories: [iot]
tags: [iot, arduino, mqtt, mosquitto, example]
image: assets/images/posts/2018-10-19/header.png
toc: true
---

Mosquitto is a message broker that implements the MQTT protocol. MQTT is very used in IOT to share info between devices. In this post we will simulate a communication between a subscriber (for example a wifi light bulb) and a publisher (a device invoking "turn on" or "turn off" the light)

## Starting MQTT Server

Create the next `docker-compose.yml` file

```yaml
version: '2.1'
services:    
  mqtt:
    image: toke/mosquitto:latest
    ports:
        - 1883:1883
        - 9001:9001
    volumes:
    - ./log/mqtt:/mqtt/log
    - ./data/mqtt:/mqtt/data/
    #- ./config/mqtt:/mqtt/config
```

This will start the mosquitto server with the default options (port 1883). If you want configure other options, uncomment the last line of docker-compose and include in that directory a mosquitto.conf file. More info of this configuration [here][mosquitto-conf]

Now, start the server with `docker-compose up`

```console
mqtt_1  | 1539943920: mosquitto version 1.4.15 (build date Sat, 07 Apr 2018 19:13:41 +0100) starting
mqtt_1  | 1539943920: Config loaded from /mqtt/config/mosquitto.conf.
mqtt_1  | 1539943920: Opening websockets listen socket on port 9001.
mqtt_1  | 1539943920: Opening ipv4 listen socket on port 1883.
mqtt_1  | 1539943920: Opening ipv6 listen socket on port 1883.
```

## Install the MQTT Mosquitto client

* Mac
`brew install mosquitto`

* Linux distributions with snap support
`snap install mosquitto`

* Others
[Download][mosquitto-download]


## Subscribe to a topic

We can simulate that we have a wifi light bulb that is listening from a topic and depending on the value it will turn on or turn off. In this case we have:

* A topic /home/light/state
* The possibles values for this topic "ON" and "OFF"

For simulate this, we must open a new tab in our console and type:

```
mosquitto_sub -t /home/light/state`
```

Now the subscriber will be listening all messages sent to the topic `/home/light/state`

## Publish to the topic

Open a new tab in your console and type:

`mosquitto_pub -t /home/light/state -m "ON"`
`mosquitto_pub -t /home/light/state -m "OFF"`
`mosquitto_pub -t /home/light/state -m "ON"`

In the terminal of previous subscriber you will can see the receive values:

```console
➜  ~ mosquitto_sub -t topic/state
ON
OFF
ON
```

## References

[1] Link to the project in [Github][github-link]


[mosquitto-conf]: https://mosquitto.org/man/mosquitto-conf-5.html
[mosquitto-download]: https://mosquitto.org/download/
[github-link]: https://github.com/frandorado/iot-projects/tree/master/mosquitto-example
