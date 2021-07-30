---
title: "RabbitMQ poisoned messages handling"
date: 2016-02-09T22:59:03+03:00
draft: false
Summary: "You never know what will come from remote system. Even if you keep everything under control, there always can be something unexpected. In microservice architecture each component must be ready for everything."
---

## 1\. The Problem

You never know what will come from remote system. Even if you keep everything under control, there always can be something unexpected. In microservice architecture each component must be ready for everything.  
When your software architecture involves message queue broker (like RabbitMQ) the interaction between your micro services looks like follows:

![](/images/rabbitmq-poisoned-messages-handeling/message-queue.png)

The problem is that if message consumer is not able to handle message, the message will be returned to queue broker and then will be delivered to the message consumer once again for the second attempt.

Here are the reasons why consumer can fail during message handling:

*   Message is corrupted (bad format)
*   Message tells lie (external resources that should be related to the message are invalid or don’t exist)
*   Consumer crash due to internal logic bug
*   Consumer crash due to external communication issue (networking, database, etc.)
*   Consumer manual restart (planned)

At first glance, all these issues must be caught and handled in one way or another. We can also have general mechanism that gives each individual message some retries count to be handled by consumer. And then if consumer still cannot handle the message we must remove the message from the working queue and put it somewhere for detailed analysis.

## 2\. General Requirements

To give each individual message some kind of ‘retries counter’, we have to store this counter value somewhere for each individual message. Once this counter value exceeds maximum allowed value, the message must be removed from the working queue.

## 3\. Documentation Overview

### 3.1\. No Such Functionality

As described [on Stack Overflow](http://stackoverflow.com/questions/23158310/how-do-i-set-a-number-of-retry-attempts-in-rabbitmq), there is no such functionality in RabbitMQ. This question was also [discussed in RabbitMQ letters](http://lists.rabbitmq.com/pipermail/rabbitmq-discuss/2013-January/025020.html), [blogs](http://kjnilsson.github.io/blog/2014/01/30/spread-the-poison/) and [forums](http://rabbitmq.1065348.n5.nabble.com/dealing-with-poison-pill-messages-td1548.html).

### 3.2\. Dead Letter Exchange

As part of queue configuration, there is an ability to set ‘dead letter exchange’. [RabbitMQ documentation](https://www.rabbitmq.com/dlx.html) describes the purpose of this feature as follows:  
Messages from a queue can be 'dead-lettered'; that is, republished to another exchange when any of the following events occur:  
•    The message is rejected (basic.reject or basic.nack) with requeue=false,  
•    The TTL for the message expires; or  
•    The queue length limit is exceeded.

## 4\. Selected Strategy

### 4.1\. Point Of View

The general requirement is to store retries counter somewhere. There are 2 places where this value can be stored:

*   In message itself
*   Not in message

First option is preferred because in this case we do not need any external storage that must support distributed access for all consumers.

### 4.2\. Incoming Message Structure

Each message contains message body and message headers. Message body is the specific information the message was created for. Message header is usually system information that allows message brocker to route messages and store other "non-user" information. So the best place to store retries counter (as I can see it) is **message headers** that are string – object map. These headers are pretty much the same as HTTP headers. So the Idea is to have something like this: **x-delivery-attempts: 3**.

### 4.3. Solution

Here is the component diagram that shows the general Idea

Here is the algorithm for different scenarios:

**Scenario A: Works as expected**

1.  Producer puts message to queue
2.  Consumer gets message from queue
3.  Consumer handles message
4.  Consumer acknowledges message
5.  Queue removes the message

**Scenario B: Poisoned message**

1.  Producer puts message to queue
2.  Consumer gets message from queue
3.  Consumer checks if x-delivery-attempt by > {max allowed retries}
4.  Consumer fails handling message but not crashes
5.  Consumer creates message copy
6.  Consumer increments x-delivery-attempt by 1 and sets this header to message copy
7.  Consumer puts message copy to the tail of queue 
8.  Consumer acknowledges original message
9.  Queue removes the original message
10.  Repeat steps 2-8 {max allowed retries} - 1 times
11.  Consumer gets message from queue
12.  Consumer checks if x-delivery-attempt by > {max allowed retries}
13.  Consumer rejects the message with require = false
14.  Queue marks message as ‘x-death’
15.  Queue moves message to dead-letter-exchange

**Scenario C: Consumer crushes unexpectedly**

1.  Producer puts message to queue
2.  Consumer gets message from queue
3.  Consumer crushes
4.  Queue detects connectivity break to consumer
5.  Queue marks message with Redelivery = true
6.  Consumer gets message from queue
7.  Consumer detects redelivery = true
8.  Consumer creates message copy
9.  Consumer increments x-delivery-attempt by 1 and set this header to message copy
10.  Consumer puts message copy to the tail of queue
11.  Scenario A or Scenario B

**Scenario C explanation**:  
Consumer should not handle redelivered messages because previous attempt finished unexpectedly and there is no guarantee it will not happen again.

### 4.4\. Dead-Letter-Exchange Configuration

As described in RabbitMQ documentation, there is an ability to configure this option using console mode or via administrative web-portal. The queue will detect dead letters automatically based on TTL, Max-length, or consumer rejects with require argument = false

### ![](/images/rabbitmq-poisoned-messages-handeling/x-dead-letter-exchange.png)

### 4.5\. Limitation

As you can see, each time when something goes wrong, the message goes to the tail of queue. 

*   Thus, message order breaks and this solution can be unacceptable if you need the message order.
*   For case of long queue and time-consuming operations, the message will reach the consumer once again with significant delay. This also can be unacceptable for some systems.