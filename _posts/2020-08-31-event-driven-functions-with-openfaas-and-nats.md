---
title: "Event-driven functions with OpenFaaS and NATS"
description: Vivek outlines how you can make use of NATS to trigger your functions in OpenFaaS through the new connector-sdk component.
date: 2020-08-31
image: /images/kafka-connector/aluminium-building.jpg
categories:
  - NATS
  - tutorial
  - examples
author_staff_member: vivek
dark_background: true
---

In this blog post, I will show how you can invoke OpenFaaS function in response to messages sent on NATS topics in publish-subscribe model.
OpenFaaS functions are accessible over HTTP endpoints via gateway service but OpenFaaS provides several other way to invoke OpenFaaS functions with help of the [connector-sdk](https://github.com/openfaas-incubator/connector-sdk).The connector sdk provides a reusable and tested interface and implementation that allows developers to quickly create a new event source. The code that is unique to OpenFaaS is standardized so that the develop can focus on the details of the receiving message from the event source.

* [kafka-connector](https://github.com/openfaas-incubator/kafka-connector) connects OpenFaaS functions to Kafka topics.
* [nats-connector](https://github.com/openfaas-incubator/nats-connector) an OpenFaaS event-connector to trigger functions from NATS. 
* [mqtt-connector](https://github.com/openfaas-incubator/mqtt-connector) MQTT connector for OpenFaaS.
* [cron-connector](https://github.com/openfaas-incubator/cron-connector) triggers OpenFaaS functions based on cron events. 
* [VMware vCenter connector](https://github.com/openfaas-incubator/openfaas-vcenter-connector) an OpenFaaS event-connector built to consume events from vCenter and to trigger functions.

There are several other connectors which allows you to trigger OpenFaaS functions based on events, are written and managed as a third party project. Please refer to this [link](https://docs.openfaas.com/reference/triggers/) for full list.

## NATS

[NATS](https://nats.io) is a simple, secure and high performance open source messaging system for cloud native applications, IoT messaging, and microservices architectures. In this blog post, I will be using NATS publish-subscribe concept where publishers publishes a message on a topic/subject and subscribers consumes that message by subscribing to that subject (sometimes called topics in other event systems).

![NATS Publish Subscribe](/images/2020-openfaas-nats/nats-publish-subscribe.png "https://docs.nats.io/nats-concepts/pubsub")

## Prerequisites

Before we start we need a couple of tools to help us quickly set up our environment:

* [docker](https://docs.docker.com/get-docker/) - required to use KinD
* [kind](https://kind.sigs.k8s.io/docs/user/quick-start/) - a tool for running local Kubernetes clusters using Docker
* [arkade](https://github.com/alexellis/arkade) - official installer for OpenFaaS, that supports many other applications for Kubernetes

> Please make sure you have these tools installed before you proceed next.

## Install Components

#### Create cluster

First thing we need is running Kubernetes cluster:

```bash
kind create cluster
```

Wait for the installation to finish run this command to verify that the cluster is ready:

```bash
kubectl -n kube-system rollout status deployment/coredns
```

#### Install OpenFaaS 
Install OpenFaaS using `arkade`

```
arkade install openfaas
```

#### Install NATS connector using arkade
`nats-connector` is connector which invokes OpenFaaS function when a message is sent to a NATS topic.

```
arkade install nats-connector 
```

> NATS comes with OpenFaaS installation, so we are not setting up NATS here

#### Install kubectl and faas-cli

install `kubectl` and `faas-cli` using `arkade` if they are not already installed.   

```
arkade get kubectl
```

```
arkade get faas-cli
```

#### Login to OpenFaaS gateway using CLI

Port forward the gateway service to access it. 

```
kubectl port-forward -n openfaas svc/gateway 8080:8080 &
```

Get the password

```
PASSWORD=$(kubectl get secret -n openfaas basic-auth -o jsonpath="{.data.basic-auth-password}" | base64 --decode; echo)
```

Login to the gateway

```
echo -n $PASSWORD | faas-cli login --username admin --password-stdin
```


## Deploy functions
`nats-connector` repo comes with test functions to verify installation. It has two OpenFaas functions.

`receive-message` function subscribes to NATS topic nats-test. On every new message on nats-test topic, receive-message will be invoked.

`publish-message` function publishes message to nats-test topic.

Deploy functions

```
faas-cli deploy -f https://raw.githubusercontent.com/openfaas-incubator/nats-connector/master/contrib/test-functions/stack.yml --read-template=false
```

> Note: You can also build and deploy this function using the stack.yml and code present in nats-connector repository.

## Verify the installation
Invoke `publish-message` function to publish a test message

```
echo  "test message" | faas-cli invoke publish-message
```

When `publish-message` was invoked, it would have pushed `test-message` to the `nats-test` topic on NATS, which would have invoked `receive-message`. We can verify that by checking logs of `receive-message` function.

```
faas-cli logs receive-message

2020-05-29T15:41:17Z 2020/05/29 15:41:17 Started logging stderr from function.
2020-05-29T15:41:17Z 2020/05/29 15:41:17 Started logging stdout from function.
2020-05-29T15:41:17Z Forking - ./handler []
2020-05-29T15:41:17Z 2020/05/29 15:41:17 OperationalMode: http
2020-05-29T15:41:17Z 2020/05/29 15:41:17 Timeouts: read: 10s, write: 10s hard: 10s.
2020-05-29T15:41:17Z 2020/05/29 15:41:17 Listening on port: 8080
2020-05-29T15:41:17Z 2020/05/29 15:41:17 Writing lock-file to: /tmp/.lock
2020-05-29T15:41:17Z 2020/05/29 15:41:17 Metrics listening on port: 8081
2020-05-29T15:42:24Z 2020/05/29 15:42:24 stderr: 2020/05/29 15:42:24 received "test message"
2020-05-29T15:42:24Z 2020/05/29 15:42:24 POST / - 200 OK - ContentLength: 28
```

## What Next ?
* Do you have a cool "event-driven" use case you want to share? Let us know and become a guest blogger!

* If you are new to OpenFaaS, I would recommend you trying [OpenFaaS workshop](https://github.com/openfaas/workshop) .

* If you don't find connector for messaging platform you are using, checkout the [connector-sdk](https://github.com/openfaas-incubator/connector-sdk) which allows you to build event-connectors for OpenFaaS.

* If you are looking to contribute to open source project, please join OpenFaaS community [slack channel](https://docs.openfaas.com/community/) and start contributing.
