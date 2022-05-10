# Consumers

Consumers can be conceived as 'views' into a stream, with their own 'cursor'. Consumers iterate or consume over all or a subset of the messages stored in the stream, according to their 'subject filter' and 'replay policy', and can be used by one or multiple client applications. It's ok to define thousands of these pointing at the same Stream.

Consumers can either be `push` based where JetStream will deliver the messages as fast as possible \(while adhering to the rate limit policy\) to a subject of your choice or `pull` to have control by asking the server for messages. The choice of what kind of consumer to use depends on the use-case but typically in the case of a client application that needs to get their own individual replay of messages from a stream you would use an 'ordered push consumer', while in the case of scaling horizontally the processing of messages from a stream you would use a 'pull consumer'.

The rate of message delivery in both cases is subject to `ReplayPolicy`. A `ReplayInstant` Consumer will receive all messages as fast as possible while a `ReplayOriginal` Consumer will receive messages at the rate they were received, which is great for replaying production traffic in staging.

In the orders example above we have 3 Consumers. The first two select a subset of the messages from the Stream by specifying a specific subject like `ORDERS.processed`. The Stream consumes `ORDERS.*` and this allows you to receive just what you need. The final Consumer receives all messages in a `push` fashion.

Consumers track their progress, they know what messages were delivered, acknowledged, etc., and will redeliver messages they sent that were not acknowledged. When first created, the Consumer has to know what message to send as the first one. You can configure either a specific message in the set \(`StreamSeq`\), specific time \(`StartTime`\), all \(`DeliverAll`\) or last \(`DeliverLast`\). This is the starting point and from there, they all behave the same - delivering all of the following messages with optional Acknowledgement.

Acknowledgements default to `AckExplicit` - the only supported mode for pull-based Consumers - meaning every message requires a distinct acknowledgement. But for push-based Consumers, you can set `AckNone` that does not require any acknowledgement, or `AckAll` which quite interestingly allows you to acknowledge a specific message, like message `100`, which will also acknowledge messages `1` through `99`. The `AckAll` mode can be a great performance boost.

Some messages may cause your applications to crash and cause a never ending loop forever poisoning your system. The `MaxDeliver` setting allow you to set an upper bound to how many times a message may be delivered.

To assist with creating monitoring applications, one can set a `SampleFrequency` which is a percentage of messages for which the system should sample and create events. These events will include delivery counts and ack waits.

When defining Consumers the items below make up the entire configuration of the Consumer:

## AckPolicy

How messages should be acknowledged. If an ack is required but is not received within the AckWait window, the message will be redelivered.

> IMPORTANT
>
> The server may consider an ack arriving out of the window. If a first process fails to ack within the window it's entirely possible, for instance in queue situation, that the message has been redelivered to another consumer. Since this will technically restart the window, the ack from the first consumer will be considered.

### AckExplicit

This is the default policy. It means that each individual message must be acknowledged. It is the only allowed option for pull consumers.

### AckNone

You do not have to ack any messages, the server will assume ack on delivery.

### AckAll

If you receive a series of messages, you only have to ack the last one you received. All the previous messages received are automatically acknowledged.

## AckWait

Ack Wait is the time in nanoseconds that the server will wait for an ack for any individual message. If an ack is not received in time, the message will be redelivered.

## DeliverPolicy / OptStartSeq / OptStartTime

When a consumer is first created, it can specify where in the stream it wants to start receiving messages. This is the `DeliverPolicy` and it's options are as follows:

### DeliverAll

All is the default policy. The consumer will start receiving from the earliest available message.

### DeliverLast

When first consuming messages, the consumer will start receiving messages with the last message added to the stream, so the very last message in the stream when the server realizes the consumer is ready.

### DeliverLastPerSubject

When first consuming messages, start with the latest one for each filtered subject currently in the stream.

### DeliverNew

When first consuming messages, the consumer will only start receiving messages that were created after the consumer was created.

### DeliverByStartSequence

When first consuming messages, start at this particular message in the set. The consumer is required to specify `OptStartSeq`, the sequence number to start on. It will receive the closest available message moving forward in the sequence should the message specified have been removed based on the stream limit policy.

### DeliverByStartTime

When first consuming messages, start with messages on or after this time. The consumer is required to specify `OptStartTime`, the time in the stream to start at. It will receive the closest available message on or after that time.

## DeliverySubject

The delivery subject is used by NATS to orchestrate how messages reach subscriptions. When a message is published to a JetStream, NATS will know of all consumers that have registered an interested in that message. It will then deliver the message to each consumer via their `DeliverySubject`. In this sense, a `DeliverySubject` can be considered like an "inbox" for each subscription.

For this reason, `DeliverySubject` is not allowed for pull subscriptions, since in that case the subscription is fetching the messages for itself. Similarly, a delivery subject is required for all push subscriptions, as well as for queue subscribing as it configures a subject that all the queue consumers should listen on.

The delivery subject should not be confused with the stream's subject. In fact, if a delivery subject had the same value as a stream's subject, it could create an infinite loop as NATS would re-deliver messages to the same subjects again and again. NATS will prevent the creation of this by emitting the following error: `Consumer creation failed: consumer deliver subject forms a cycle (10081)`.

If two consumers share the same `DeliverySubject` then the messages will be delivered to both, since subscriptions will be bound to the same subject. In cases in which you want a subscription to receive all (filtered) messages of a stream, a unique `DeliverySubject` should be provided.

## Durable \(Name\)

The name of the Consumer, which the server will track, allowing resuming consumption where left off. By default, a consumer is ephemeral. To make the consumer durable, set the name.

## FilterSubject

When consuming from a stream with a wildcard subject, this allows you to select a subset of the full wildcard subject to receive messages from.

## MaxAckPending

MaxAckPending implements a simple form of _one-to-many_ flow control. It sets the maximum number of messages without an acknowledgement that can be outstanding, once this limit is reached message delivery will be suspended. It cannot be used with AckNone ack policy. This maximum number of pending acks applies for _all_ of the consumer's subscriber processes. A value of -1 means there can be any number of pending acks (i.e. no flow control).

### Note about push and pull consumers: 

The MaxAckPending's one-to-many flow control functionality is only useful for push consumers. For pull consumers you should disable it (i.e. -1), as it can otherwise place a limit on the horizontal scalability of the processing of the stream. Because delivery of the messages to the client application through pull consumers is client demand-driven rather than server initiated, there is no need for any kind of one-to-many flow control. With pull consumers at a given point in time, the number of pending acks is a function of the number of client applications calling `fetch` on the pull consumer and the requested batch size for that operation. 

## FlowControl

This flow control setting is to enable or not another form of flow control in parallel to MaxAckPending. But unlike MaxAckPending it is a _one-to-one_ flow control that operates independently for each individual subscriber to the consumer. It uses a sliding-window flow-control protocol whose attributes (e.g. size of the window) are _not_ user adjustable.

## IdleHeartbeat

If the idle heartbeat period is set, the server will regularly send a status message to the client (i.e. when the period has elapsed) while there are no new messages to send. This lets the client know that the JetStream service is still up and running, even when there is no activity on the stream. The message status header will have a code of 100. Unlike FlowControl, it will have no reply to address. It may have a description like "Idle Heartbeat"

## MaxDeliver

The maximum number of times a specific message will be delivered. Applies to any message that is re-sent due to ack policy.

## RateLimit

Used to throttle the delivery of messages to the consumer, in bits per second.

## ReplayPolicy

The replay policy applies when the DeliverPolicy is `DeliverAll`, `DeliverByStartSequence` or `DeliverByStartTime` since those deliver policies begin reading the stream at a position other than the end. If the policy is `ReplayOriginal`, the messages in the stream will be pushed to the client at the same rate that they were originally received, simulating the original timing of messages. If the policy is `ReplayInstant` \(the default\), the messages will be pushed to the client as fast as possible while adhering to the Ack Policy, Max Ack Pending and the client's ability to consume those messages.

## SampleFrequency

Sets the percentage of acknowledgements that should be sampled for observability, 0-100 This value is a string and for example allows both `30` and `30%` as valid values.

