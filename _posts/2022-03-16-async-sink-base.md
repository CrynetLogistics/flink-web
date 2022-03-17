---
layout: post
title: "Asynchronous Base Sink"
date: 2022-03-17 16:00:00
authors:
- CrynetLogistics:
  name: "Zichen Liu"
  twitter: "CrynetLogistics"
excerpt: An overview of the new features of the new Async Base Sink and pointers for building your own concrete sink atop
---

The basic functionalities of sinks in general are quite similar. They batch records according to user defined buffering hints, sign requests, write them to the destination, retry unsuccessful or throttled requests, and participate in checkpointing.

New for [Flink 1.15](https://cwiki.apache.org/confluence/display/FLINK/FLIP-171%3A+Async+Sink) is the Async Base Sink - an abstract sink with a number of common functionalities extracted. Adding support for a new destination now only requires a lightweight shim that implements the specific interfaces of the destination using a client that supports async requests. 

This common abstraction will reduce the effort required to maintain all these individual sinks, with bugfixes and improvements to the sink core benefiting all implementations that extend it.

**Attention** The sink is designed to participate in checkpointing to provide at-least once semantics, but it is limited to destinations that provide a client that supports async requests.

The design of the sink focuses on extensibility and a broad support of destinations. The core of the sink is kept generic and free of any connector specific dependencies.


{% toc %}

# Dependency
To use this base sink, add the following dependency to your project:

```xml
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-connector-base</artifactId>
  <version>${flink.version}</version>
</dependency>
```



# Public Interfaces

## Generic Types

`<InputT>` – type of elements in a DataStream that should be passed to the sink

`<RequestEntryT>` – type of a payload containing the element and additional metadata that is required to submit a single element to the destination


## Element Converter Interface
```java
public interface ElementConverter<InputT, RequestEntryT> extends Serializable {
    RequestEntryT apply(InputT element, SinkWriter.Context context);
}
```
Concrete sink implementers should provide a way to convert from an element in the DataStream to the payload type that contains all the additional metadata required to submit that element to the destination by the sink.
Ideally this would be hidden from the end user as it allows concrete sink implementers to adapt to changes in the destination api without breaking end user code. 


## Sink Writer Interface

```java
public abstract class AsyncSinkWriter<InputT, RequestEntryT extends Serializable>
        implements SinkWriter<InputT, Void, Collection<RequestEntryT>> {
    // ...
    protected abstract void submitRequestEntries
            (List<RequestEntryT> requestEntries, Consumer<List<RequestEntryT>> requestResult);
    // ...
}
```

This method should specify how a list of elements from the datastream may be persisted into the destination. Sink implementers of various datastore and data processing vendors may use their own clients in connecting to and persisting the requestEntries received by this method.

Should any elements fail to be persisted, they should be requeued back in the buffer for retry using `requestResult.accept(...list of failed entries...)`. However, retrying any element that is known to be faulty and consistently failing, will result in that element being requeued forever, therefore a sensible strategy for determining what should be retried is highly recommended.

```java
public abstract class AsyncSinkWriter<InputT, RequestEntryT extends Serializable>
        implements SinkWriter<InputT, Void, Collection<RequestEntryT>> {
    // ...
    protected abstract long getSizeInBytes(RequestEntryT requestEntry);
    // ...
}
```
The generic sink has a concept of size of elements in the buffer. This allows users to specify a byte size threshold beyond which elements will be flushed. However the sink implementer is best positioned to determine what is most sensible measure of size for each `RequestEntryT` is.


```java
public abstract class AsyncSinkWriter<InputT, RequestEntryT extends Serializable>
        implements SinkWriter<InputT, Void, Collection<RequestEntryT>> {
    // ...
    public AsyncSinkWriter(
            ElementConverter<InputT, RequestEntryT> elementConverter,
            Sink.InitContext context,
            int maxBatchSize,
            int maxInFlightRequests,
            int maxBufferedRequests,
            long flushOnBufferSizeInBytes,
            long maxTimeInBufferMS,
            long maxRecordSizeInBytes) { /* ... */ }
    // ...
}
```

## Sink Interface

```java
public class MySink<InputT> extends AsyncSinkBase<InputT, RequestEntryT> {
    // ...
    @Override
    public SinkWriter<InputT, Void, Collection<PutRecordsRequestEntry>> createWriter
            (InitContext context, List<Collection<PutRecordsRequestEntry>> states) {
        return new MySinkWriter(context);
    }
    // ...
}
```
Sink implementers extending this will need to return their own extension of the `AsyncSinkWriter` from `createWriter()` inside their own implementation of `AsyncSinkBase`.

Currently the [Kinesis Data Streams sink](https://github.com/apache/flink/tree/master/flink-connectors/flink-connector-aws-kinesis-streams) and [Kinesis Data Firehose sink](https://github.com/apache/flink/tree/master/flink-connectors/flink-connector-aws-kinesis-firehose) are using this base sink. 

# Metrics

There are 3 metrics that implementing sinks will automatically benefit from and therefore shouldn't implement themselves.

* CurrentSendTime Gauge - returns the amount of time in milliseconds it took for the most recent request to write records to return, whether successful or not.  
* NumBytesOut Counter - counts the total number of bytes the sink has tried to write to the destination, using the method `getSizeInBytes` to determine the size of each record. This will double count failures that may need to be retried. 
* NumRecordsOut Counter - similar to above, this counts the total number of records the sink has tried to write to the destination. This will double count failures that may need to be retried.

# Sink Behaviour


There are 6 sink configuration settings that control the buffering, flushing and retry behaviour of the sink.

* `int maxBatchSize` - maximum number of elements that may be passed in the   list to submitRequestEntries to be written downstream.
* `int maxInFlightRequests` - maximum number of uncompleted calls to submitRequestEntries that the SinkWriter will allow at any given point. Once this point has reached, writes and callbacks to add elements to the buffer may block until one or more requests to submitRequestEntries completes.
* `int maxBufferedRequests` - maximum buffer length. Callbacks to add elements to the buffer and calls to write will block if this length has been reached and will only unblock if elements from the buffer have been removed for flushing.
* `long flushOnBufferSizeInBytes` - a flush will be attempted if the most recent call to write introduces an element to the buffer such that the total size of the buffer is greater than or equal to this threshold value.
* `long maxTimeInBufferMS` - maximum amount of time an element may remain in the buffer. In most cases elements are flushed as a result of the batch size (in bytes or number) being reached or during a snapshot. However, there are scenarios where an element may remain in the buffer forever or a long period of time. To mitigate this, a timer is constantly active in the buffer such that: while the buffer is not empty, it will flush every maxTimeInBufferMS milliseconds.
* `long maxRecordSizeInBytes` - maximum size in bytes allowed for a single record, as determined by `getSizeInBytes()`.

Destinations typically have a defined throughput limit and will begin throttling or rejecting requests once near. With multiple subtasks, we employ [Additive Increase Multiplicative Decrease (AIMD)](https://en.wikipedia.org/wiki/Additive_increase/multiplicative_decrease) as a strategy for selecting the optimal batch size.
