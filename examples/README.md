Stream examples
===

 - [Getting started](./getting_started.go). A good point to start.
 - [Offset Start](./offsetStart/offset.go). How to set different points to start consuming
 - [Offset Tracking](./offsetTracking/offsetTracking.go). Manually store the consumer offset 
 - [Automatic Offset Tracking](./automaticOffsetTracking/automaticOffsetTracking.go). Automatic store the consumer offset
 - [Getting started TLS](./tls/getting_started_tls.go). A TLS example. ( you can run `make rabbitmq-server-tls` to create a tls single rabbitmq node )
 - [HA Producer](./haProducer/producer.go). HA producer example (Still experimental)
 - [Deduplication](./deduplication/deduplication.go). deduplication example, run it more than one time, and the records <br />
   won't change, since the server will handle the deduplication.
 - [Using a load balancer](./proxy/proxy.go). An example how to use the client with a TLS load balancer.<br />
   Use the [RabbitMQ TLS cluster](../compose) to run a TLS and no TLS cluster. <br />
   For more details: https://blog.rabbitmq.com/posts/2021/07/connecting-to-streams/
 - [Sub Entries Batching](./sub-entries-batching/sub_entries_batching.go). Sub Entries Batching example
