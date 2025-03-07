<h1 align="center">RabbitMQ Stream GO Client</h1>

---
<div align="center">

![Build](https://github.com/rabbitmq/rabbitmq-stream-go-client/workflows/Build/badge.svg)
[![codecov](https://codecov.io/gh/rabbitmq/rabbitmq-stream-go-client/branch/main/graph/badge.svg?token=HZD4S71QIM)](https://codecov.io/gh/rabbitmq/rabbitmq-stream-go-client)

Go client for [RabbitMQ Stream Queues](https://github.com/rabbitmq/rabbitmq-server/tree/master/deps/rabbitmq_stream)
</div>

# Table of Contents

- [Overview](#overview)
- [Installing](#installing)
- [Run server with Docker](#run-server-with-docker)
- [Getting started for impatient](#getting-started-for-impatient)
- [Examples](#examples)
- [Usage](#usage)
    * [Connect](#connect)
        * [Multi hosts](#multi-hosts)
        * [Load Balancer](#load-balancer)
        * [TLS](#tls)
      * [Streams](#streams)
		* [Statistics](#streams-statistics)
    * [Publish messages](#publish-messages)
        * [`Send` vs `BatchSend`](#send-vs-batchsend)
        * [Publish Confirmation](#publish-confirmation)
        * [Deduplication](#deduplication)
        * [Sub Entries Batching](#sub-entries-batching)
        * [HA producer - Experimental](#ha-producer-experimental)
    * [Consume messages](#consume-messages)
        * [Manual Track Offset](#manual-track-offset)
        * [Automatic Track Offset](#automatic-track-offset)
        * [Get consumer Offset](#get-consumer-offset)
    * [Handle Close](#handle-close)
- [Performance test tool](#performance-test-tool)
    * [Performance test tool Docker](#performance-test-tool-docker)
- [Build form source](#build-form-source)
- [Project status](#project-status)

### Overview

Go client for [RabbitMQ Stream Queues](https://github.com/rabbitmq/rabbitmq-server/tree/master/deps/rabbitmq_stream)

### Installing

```shell
 go get -u github.com/rabbitmq/rabbitmq-stream-go-client
```

imports:
```golang
"github.com/rabbitmq/rabbitmq-stream-go-client/pkg/stream" // Main package
"github.com/rabbitmq/rabbitmq-stream-go-client/pkg/amqp" // amqp 1.0 package to encode messages
"github.com/rabbitmq/rabbitmq-stream-go-client/pkg/message" // messages interface package, you may not need to import it directly
```

### Run server with Docker
---
You may need a server to test locally. Let's start the broker:
```shell
docker run -it --rm --name rabbitmq -p 5552:5552 -p 15672:15672\
    -e RABBITMQ_SERVER_ADDITIONAL_ERL_ARGS='-rabbitmq_stream advertised_host localhost -rabbit loopback_users "none"' \
    rabbitmq:3.9-management
```
The broker should start in a few seconds. When it’s ready, enable the `stream` plugin and `stream_management`:
```shell
docker exec rabbitmq rabbitmq-plugins enable rabbitmq_stream_management
```

Management UI: http://localhost:15672/ </br>
Stream uri: `rabbitmq-stream://guest:guest@localhost:5552`

### Getting started for impatient

See [getting started](./examples/getting_started.go) example.

### Examples

See [examples](./examples/) directory for more use cases.

# Usage

### Connect

Standard way to connect single node:
```golang
env, err := stream.NewEnvironment(
		stream.NewEnvironmentOptions().
			SetHost("localhost").
			SetPort(5552).
			SetUser("guest").
			SetPassword("guest"))
	CheckErr(err)
```

you can define the number of producers per connections, the default value is 1:
```golang
stream.NewEnvironmentOptions().
SetMaxProducersPerClient(2))
```

you can define the number of consumers per connections, the default value is 1:
```golang
stream.NewEnvironmentOptions().
SetMaxConsumersPerClient(2))
```

To have the best performance you should use the default values.
Note about multiple consumers per connection:
*The IO threads is shared across the consumers, so if one consumer is slow it could impact other consumers performances*

### Multi hosts

It is possible to define multi hosts, in case one fails to connect the clients tries random another one.

```golang
addresses := []string{
		"rabbitmq-stream://guest:guest@host1:5552/%2f",
		"rabbitmq-stream://guest:guest@host2:5552/%2f",
		"rabbitmq-stream://guest:guest@host3:5552/%2f"}

env, err := stream.NewEnvironment(
			stream.NewEnvironmentOptions().SetUris(addresses))
```

### Load Balancer

The stream client is supposed to reach all the hostnames,
in case of load balancer you can use the `stream.AddressResolver` parameter in this way:

```golang
addressResolver := stream.AddressResolver{
		Host: "load-balancer-ip",
		Port: 5552,
	}
env, err := stream.NewEnvironment(
		stream.NewEnvironmentOptions().
			SetHost(addressResolver.Host).
			SetPort(addressResolver.Port).
			SetAddressResolver(addressResolver).
```

In this configuration the client tries the connection until reach the right node.

This [rabbitmq blog post](https://blog.rabbitmq.com/posts/2021/07/connecting-to-streams/) explains the details.

See also "Using a load balancer" example in the [examples](./examples/) directory

### TLS

To configure TLS you need to set the `IsTLS` parameter:
```golang
env, err := stream.NewEnvironment(
		stream.NewEnvironmentOptions().
			SetHost("localhost").
			SetPort(5551). // standard TLS port
			SetUser("guest").
			SetPassword("guest").
			IsTLS(true).
			SetTLSConfig(&tls.Config{}),
	)
```

The `tls.Config` is the standard golang tls library https://pkg.go.dev/crypto/tls </br>
See also "Getting started TLS" example in the [examples](./examples/) directory


### Streams

To define streams you need to use the the `environment` interfaces `DeclareStream` and `DeleteStream`.

It is highly recommended to define stream retention policies during the stream creation, like `MaxLengthBytes` or `MaxAge`:

```golang
err = env.DeclareStream(streamName,
		stream.NewStreamOptions().
		SetMaxLengthBytes(stream.ByteCapacity{}.GB(2)))
```

The function `DeclareStream` doesn't return errors if a stream is already defined with the same parameters.
Note that it returns the precondition failed when it doesn't have the same parameters
Use `StreamExists` to check if a stream exists.

### Streams Statistics

To get stream statistics you need to use the the `environment.StreamStats` method.

```golang
stats, err := environment.StreamStats(testStreamName)

// FirstOffset - The first offset in the stream.
// return first offset in the stream /
// Error if there is no first offset yet

firstOffset, err := stats.FirstOffset() // first offset of the stream

// LastOffset - The last offset in the stream.
// return last offset in the stream
// error if there is no first offset yet
lastOffset, err := stats.LastOffset() // last offset of the stream

// CommittedChunkId - The ID (offset) of the committed chunk (block of messages) in the stream.
//
//	It is the offset of the first message in the last chunk confirmed by a quorum of the stream
//	cluster members (leader and replicas).
//
//	The committed chunk ID is a good indication of what the last offset of a stream can be at a
//	given time. The value can be stale as soon as the application reads it though, as the committed
//	chunk ID for a stream that is published to changes all the time.

committedChunkId, err := statsAfter.CommittedChunkId()
```

### Publish messages

To publish a message you need a `*stream.Producer` instance:
```golang
producer, err :=  env.NewProducer("my-stream", nil)
```

With `ProducerOptions` is possible to customize the Producer behaviour:
```golang
type ProducerOptions struct {
	Name       string // Producer name, it is useful to handle deduplication messages
	QueueSize  int // Internal queue to handle back-pressure, low value reduces the back-pressure on the server
	BatchSize  int // It is the batch-size aggregation, low value reduce the latency, high value increase the throughput
	BatchPublishingDelay int    // Period to send a batch of messages.
}
```

The client provides two interfaces to send messages.
`send`:
```golang
var message message.StreamMessage
message = amqp.NewMessage([]byte("hello"))
err = producer.Send(message)
```
and `BatchSend`:
```golang
var messages []message.StreamMessage
for z := 0; z < 10; z++ {
  messages = append(messages, amqp.NewMessage([]byte("hello")))
}
err = producer.BatchSend(messages)
```


`producer.Send`:
- accepts one message as parameter
- automatically aggregates the messages
- automatically splits  the messages in case the size is bigger than `requestedMaxFrameSize`
- automatically splits the messages based on batch-size
- sends the messages in case nothing happens in `producer-send-timeout`
- is asynchronous

`producer.BatchSend`:
- accepts an array messages as parameter
- is synchronous

Close the producer:
`producer.Close()` the producer is removed from the server. TCP connection is closed if there aren't </b>
other producers

### `Send` vs `BatchSend`

The `BatchSend` is the primitive to send the messages, `Send` introduces a smart layer to publish messages and internally uses `BatchSend`.

The `Send` interface works in most of the cases, In some condition is about 15/20 slower than `BatchSend`. See also this [thread](https://groups.google.com/g/rabbitmq-users/c/IO_9-BbCzgQ).

### Publish Confirmation

For each publish the server sends back to the client the confirmation or an error.
The client provides an interface to receive the confirmation:

```golang
//optional publish confirmation channel
chPublishConfirm := producer.NotifyPublishConfirmation()
handlePublishConfirm(chPublishConfirm)

func handlePublishConfirm(confirms stream.ChannelPublishConfirm) {
	go func() {
		for confirmed := range confirms {
			for _, msg := range confirmed {
				if msg.IsConfirmed() {
					fmt.Printf("message %s stored \n  ", msg.GetMessage().GetData())
				} else {
					fmt.Printf("message %s failed \n  ", msg.GetMessage().GetData())
				}
			}
		}
	}()
}
```

In the MessageStatus struct you can find two `publishingId`:
```golang
//first one
messageStatus.GetMessage().GetPublishingId()
// second one
messageStatus.GetPublishingId()
```

The first one is provided by the user for special cases like Deduplication.
The second one is assigned automatically by the client.
In case the user specifies the `publishingId` with:
```golang
msg = amqp.NewMessage([]byte("mymessage"))
msg.SetPublishingId(18) // <---
```


The filed: `messageStatus.GetMessage().HasPublishingId()` is true and </br>
the values `messageStatus.GetMessage().GetPublishingId()` and `messageStatus.GetPublishingId()` are the same.


See also "Getting started" example in the [examples](./examples/) directory



### Deduplication

The stream plugin can handle deduplication data, see this blog post for more details:
https://blog.rabbitmq.com/posts/2021/07/rabbitmq-streams-message-deduplication/ </br>
You can find a "Deduplication" example in the [examples](./examples/) directory. </br>
Run it more than time, the messages count will be always 10.

To retrieve the last sequence id for producer you can use:
```
publishingId, err := producer.GetLastPublishingId()
```

### Sub Entries Batching

The number of messages to put in a sub-entry. A sub-entry is one "slot" in a publishing frame,
meaning outbound messages are not only batched in publishing frames, but in sub-entries as well.
Use this feature to increase throughput at the cost of increased latency. </br>
You can find a "Sub Entries Batching" example in the [examples](./examples/) directory. </br>

Default compression is `None` (no compression) but you can define different kind of compressions: `GZIP`,`SNAPPY`,`LZ4`,`ZSTD` </br>
Compression is valid only is `SubEntrySize > 1`

```golang
producer, err := env.NewProducer(streamName, stream.NewProducerOptions().
		SetSubEntrySize(100).
		SetCompression(stream.Compression{}.Gzip()))
```

### Ha Producer Experimental
The ha producer is built up the standard producer. </br>
Features:
 - auto-reconnect in case of disconnection
 - handle the unconfirmed messages automatically in case of fail.

You can find a "HA producer" example in the [examples](./examples/) directory. </br>

```golang
haproducer := NewHAProducer(
	env *stream.Environment, // mandatory
	streamName string, // mandatory
	producerOptions *stream.ProducerOptions, //optional
	confirmMessageHandler ConfirmMessageHandler // mandatory
	)
```

### Consume messages

In order to consume messages from a stream you need to use the `NewConsumer` interface, ex:
```golang
handleMessages := func(consumerContext stream.ConsumerContext, message *amqp.Message) {
	fmt.Printf("consumer name: %s, text: %s \n ", consumerContext.Consumer.GetName(), message.Data)
}

consumer, err := env.NewConsumer(
		"my-stream",
		handleMessages,
		....
```

With `ConsumerOptions` it is possible to customize the consumer behaviour.
```golang
  stream.NewConsumerOptions().
  SetConsumerName("my_consumer").                  // set a consumer name
  SetCRCCheck(false).  // Enable/Disable the CRC control.
  SetOffset(stream.OffsetSpecification{}.First())) // start consuming from the beginning
```
Disabling the CRC control can increase the performances.

See also "Offset Start" example in the [examples](./examples/) directory

Close the consumer:
`consumer.Close()` the consumer is removed from the server. TCP connection is closed if there aren't </b>
other consumers

### Manual Track Offset
The server can store the current delivered offset given a consumer, in this way:
```golang
handleMessages := func(consumerContext stream.ConsumerContext, message *amqp.Message) {
		if atomic.AddInt32(&count, 1)%1000 == 0 {
			err := consumerContext.Consumer.StoreOffset()  // commit all messages up to the current message's offset
			....

consumer, err := env.NewConsumer(
..
stream.NewConsumerOptions().
			SetConsumerName("my_consumer"). <------
```
A consumer must have a name to be able to store offsets. <br>
Note: *AVOID to store the offset for each single message, it will reduce the performances*

See also "Offset Tracking" example in the [examples](./examples/) directory

The server can also store a previous delivered offset rather than the current delivered offset, in this way:
```golang
processMessageAsync := func(consumer stream.Consumer, message *amqp.Message, offset int64) {
    ....
    err := consumer.StoreCustomOffset(offset)  // commit all messages up to this offset
    ....
```
This is useful in situations where we have to process messages asynchronously and we cannot block the original message
handler. Which means we cannot store the current or latest delivered offset as we saw in the `handleMessages` function
above.

### Automatic Track Offset

The following snippet shows how to enable automatic tracking with the defaults:
```golang
stream.NewConsumerOptions().
			SetConsumerName("my_consumer").
			SetAutoCommit(stream.NewAutoCommitStrategy() ...
```
`nil` is also a valid value. Default values will be used

```golang
stream.NewConsumerOptions().
			SetConsumerName("my_consumer").
			SetAutoCommit(nil) ...
```
Set the consumer name (mandatory for offset tracking) </br>

The automatic tracking strategy has the following available settings:
- message count before storage: the client will store the offset after the specified number of messages, </br>
right after the execution of the message handler. The default is every 10,000 messages.

- flush interval: the client will make sure to store the last received offset at the specified interval. </br>
This avoids having pending, not stored offsets in case of inactivity.  The default is 5 seconds.

Those settings are configurable, as shown in the following snippet:
```golang
stream.NewConsumerOptions().
	// set a consumerOffsetNumber name
	SetConsumerName("my_consumer").
	SetAutoCommit(stream.NewAutoCommitStrategy().
		SetCountBeforeStorage(50). // store each 50 messages stores
		SetFlushInterval(10*time.Second)). // store each 10 seconds
	SetOffset(stream.OffsetSpecification{}.First()))
```

See also "Automatic Offset Tracking" example in the [examples](./examples/) directory

### Get consumer offset

It is possible to query the consumer offset using:
```golang
offset, err := env.QueryOffset("consumer_name", "streamName")
```
An error is returned if the offset doesn't exist.


### Handle Close
Client provides an interface to handle the producer/consumer close.

```golang
channelClose := consumer.NotifyClose()
defer consumerClose(channelClose)
func consumerClose(channelClose stream.ChannelClose) {
	event := <-channelClose
	fmt.Printf("Consumer: %s closed on the stream: %s, reason: %s \n", event.Name, event.StreamName, event.Reason)
}
```
In this way it is possible to handle fail-over

### Performance test tool

Performance test tool it is useful to execute tests.
See also the [Java Performance](https://rabbitmq.github.io/rabbitmq-stream-java-client/stable/htmlsingle/#the-performance-tool) tool


To install you can download the version from github:

Mac:
```
https://github.com/rabbitmq/rabbitmq-stream-go-client/releases/latest/download/stream-perf-test_darwin_amd64.tar.gz
```

Linux:
```
https://github.com/rabbitmq/rabbitmq-stream-go-client/releases/latest/download/stream-perf-test_linux_amd64.tar.gz
```

Windows
```
https://github.com/rabbitmq/rabbitmq-stream-go-client/releases/latest/download/stream-perf-test_windows_amd64.zip
```

execute `stream-perf-test --help` to see the parameters. By default it executes a test with one producer, one consumer.

here an example:
```shell
stream-perf-test --publishers 3 --consumers 2 --streams my_stream --max-length-bytes 2GB --uris rabbitmq-stream://guest:guest@localhost:5552/  --fixed-body 400 --time 10
```

### Performance test tool Docker
A docker image is available: `pivotalrabbitmq/go-stream-perf-test`, to test it:

Run the server is host mode:
```shell
 docker run -it --rm --name rabbitmq --network host \
    rabbitmq:3.9-management
```
enable the plugin:
```
 docker exec rabbitmq rabbitmq-plugins enable rabbitmq_stream
```
then run the docker image:
```shell
docker run -it --network host  pivotalrabbitmq/go-stream-perf-test
```

To see all the parameters:
```shell
docker run -it --network host  pivotalrabbitmq/go-stream-perf-test --help
```

### Build form source

```shell
make build
```

To execute the tests you need a docker image, you can use:
```shell
make rabbitmq-server
```
to run a ready rabbitmq-server with stream enabled for tests.

then `make test`
