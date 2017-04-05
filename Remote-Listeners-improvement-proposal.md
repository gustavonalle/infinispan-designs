## Background

With [ISPN-374](https://issues.jboss.org/browse/ISPN-374), remote listeners were introduced allowing Hot Rod clients 
to receive notifications of cache events happening in the cluster.
In order to receive those events, the Hot Rod client internally registers itself with a single server from the cluster, which in turn will act as the proxy for all events happening on other nodes. 

#### Fail-over

If the server where the listener was registered goes down, the Hot Rod client automatically registers itself with another server, thus providing transparent fail-over.

#### At-most-once deliver

During a fail-over, more precisely between the time the server previously holding the listener goes down and the time a new listener registration happens in another server, the events that happened while the listener was being setting up will not be replayed to the client by default, making the notification subsystem in this situation a "at-most-once" delivery.

#### At-least-once deliver

A client can configure a listener with ```includeCurrentState=true```, causing it to send the entire state of the cluster upon listener registration. Before sending the state, a fail-over event will be generated in order to signal to the client that the listener registration changed and thus allow for any client-side action.
The usage of ```includeCurrentState=true``` makes the notification subsystem 'at-least-once', since duplicated messages will be received in case of fail-over, but no state should be lost.

## Issues

#### State vs Event

The eventset that is sent when using ```includeCurrentState=true``` actually represents the cache state at the moment of the registration of the listener, so it does not accurately replay events that occurred previously in the cache. Example, consider the following history between client and servers:

REGISTER LISTENER ON SERVER1   
CACHE.PUT(K,V)   
CACHE.PUT(K,V2)  
SERVER1 GOES DOWN    <-- Begin fail-over  
CACHE.REMOVE(K)  
CACHE.PUT(K,V3)  
CACHE.PUT(K,V4)  
REGISTER LISTENER ON SERVER2  <-- End fail-over  

The client would receive two notifications before the fail-over in this scenario: CREATED(K,V) and MODIFIED(K,V2). After the fail-over another event with CREATED(K,V4) is sent since this was the state when the listener was registered for the second time. The removal of K and further change to (K,V3) would be collapsed.

This behavior is sufficient for near caches that are just interested in the state, not on 
the actually event history that led to the state. This is not true for Continuous queries where
the fact that an entry is removed from a matching query is meaningfull.

#### Performance
For large caches and large number of listeners, the includeCurrentState can be overkill. 


## The proposal



### Event storage

In order to broaden usage scope of remote listener by addressing the above issues, a storage of events
would be required. 
  
Each cache would be associated with an off-heap, per segment and bounded event storage that would persist the events with a monotonically increasing id. 

The storage time frame is configurable; the aim is not to have an unbounded storage but to provide a 
suitable value to give guarantees for a particular use case. Long term storage is better handled by a time series database.

#### Monotonically increasing id

In order to avoid central point of contention, the use of [Flake Ids](http://yellerapp.com/posts/2015-02-09-flake-ids.html) is an option. The Flake Ids are interesting because they don't need a coordinator, and to validate their applicability, the following constraints are needed:

* All Ids generated by a single primary owner need to be ordered
* Ids generated across primary owners can be roughly ordered
* The initial coordination (if needed) should take into account that the cluster size is not fixed.

#### Storage granularity

Event storage is cache scoped, per segment and ordered. This implies that operations for a single key are ordered.

#### Storage provider

The storage can be pluggable, and the default provider is a Lucene index, given:

* It is already "native" to Infinispan
* It is persistent and memory mapped 
* It is efficient to query by id range
* It is heavily optimized for immutable data
* It has been used in this scenario successfully. Examples [here](http://qaware.blogspot.co.uk/2015/02/apache-solr-as-compressed-scalable-and.html) and [here](https://www.elastic.co/blog/elasticsearch-as-a-time-series-data-store)

#### Storage replication

The event storage needs to be replicated, and this can be handled by a custom interceptor that will write events in the primary and backup owners. 

QUESTION: The event id is generated in the PRIMARY_OWNER, and the replicas cannot generate a new one. Would it be possible to propagate the eventId generate in the primary to the replicas?

#### Storage rebalancing

Rebalancing needs to be supported as well. During rebalancing, for each migrated segment, the new owner will obtain the related events from the old owner, and the old owner will delete from its local storage the migrated events.

### Serving events

A specialized cache level component would be responsible to serve events by segment and by id range

#### By Segment

Used for state transfer. This request will arrive during state transfer to segment owners, and it involves returning the local events associated with a particular segment.

#### By Id range

Used by consumers. Any node in the cluster will be able to serve events by id range. In order to do that, it will broadcast a Lucene query across the cluster, and aggregate the results before serving. Although it is a two phase query, it is expected to perform well, since:

* On every node a near-real-time reader will be opened
* The Lucene directory will be memory mapped by the OS in each node
* Query is done in parallel across all nodes

### Consumers

Consumers would still register with the server in order to:

* Specify a filter
* Specify a query
 
and they would received back:

* the listener Id
* the last id in the event log


The client would contact any server in the cluster to obtain events, passing back the ```listenerId``` and ```lastId``` and receive the events or a filtered subset of it.

The client would then store locally the id of the last event consumed/received and would ask the server 
for the next events, passing back the last acknowledged id. The client can also be configured with a batch size (or a poll frequency) to control the events polling frequency.

This pull style architecture will allow the client to consume messages in its own pace, and if the client dies and comes back later, as long as the last id consumed is available, it can resume consuming messages from where it stopped.

#### Obtaining the Initial state

There is already a mechanism to obtain the initial state of a cache in a pull style, which is the remote iterator, so no need for the server to push down all the cache entries.

#### Listener registry

At the moment, each consumer is connected to the same initially chosen server during the push style event consumption. 

This means that a particular server keeps tracks of the listenerIds and associated filters only for its own consumers. 
If the consumers were to be free to contact any server and supply the lastId, this registry would need to be moved to a replicated cache.

#### Consumer window

If the customer disconnects and reconnects after a time period which is larger than the configured event storage retention, it will loose events, and its only choice is to ask for the state of the cache.

#### Last Id storage

Infinispan would be not responsible for the lastId storage in the client side. It's the application responsibility to store it properly so that it can survives crashes. If the JVM process with the listener crashes but the last Id is stored on disk, the consumption can be resumed later. If the whole container/server where the listener resides crashes, it'd loose the last id and thus the capacity to resume events. To prevent this, it'd need to use a 3rd party storage such as a shared volume or a database.

#### Server initiated poll

The consumer would be configurable with the poll interval, and a batch size. For situations that a cache rarely changes and polling is wasteful, a consumer could be connected to the server and receive a notification that the event log changed, and then resume with the normal pull.

## Consumer checklist

The following consumers need to be supported/benefit from this proposal:

* JCache 
* Clustered Queries (Java and non-Java)
* Remote listeners (Java and non-Java)
* Spark connector (DStream)
* Near caches
* Debezium
* Other eventual new integrations: Apache Flick, Storm, etc

## Open questions

* JCache needs sync listeners. How would this work with this proposal? Need to check if the remote JCache implementation really needs this or if is doing an emulated sync layer on top of async listeners.
* Assuming for the case where all clients are regular clients (no listeners need), maitaining an event log is overkill. Should this be disabled by default? 
* Should the push model be maintained as an alternative?