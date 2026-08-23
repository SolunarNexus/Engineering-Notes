> From high-level it is an event streaming platform used either as a message queue or a stream processing system.

**Producer** - entity responsible for publishing events in the queue (server, process, ...)

**Consumer** - entity taking event from the queue and changing its behavior (website, ...)

**Consumer group** - group of consumers where it is guaranteed that an event is processed by exactly one consumer from the group.

**Broker** - the servers (physical or virtual) that hold the event partitions (queues). Each can have multiple partitions.

**Partition** - the queue. An *ordered*, *immutable* sequence of messages that we append to (like a log file). Messages usually belong to the same logical origin (e.g. a particular football game). It's a physical grouping of messages - a way of scaling data.

**Topic** - a logical grouping of partitions - a way of organizing data (e.g. topic - football and partitions - football games or topic - basketball and partitions - basketball games). Producer publishes to, and consumers consumes from topic.

**Message/Record** - an event (from Kafka terminology). The item actually stored in a partition. Consists of four parts:
1. *Header* - some additional information such as HTTP headers
2. *Key* - partition key
3. *Value* - arbitrary payload
4. *Timestamp* - determines the ordering withing partition. If none is specified, the machine (broker) time is used instead

**Message publishing mechanism**:
1. Publish message to a topic
2. If there's a partition key present, it's hash is used to determine the partition. I believe the MurMurHash is used and the operation looks like `hash(key) % N` where `N` is the number of partitions. There's a controller in Kafka cluster that has a mapping of brokers and partitions.
3. Then the broker for partition is identified
4. Send message to broker
5. Broker appends the message to the correct partition for us.
> **note**: if the partition key is not present, the partition is selected using Round-Robin or other logic. Keys are optional. However, not adding a key does not scale.

**Message consuming mechanism**:
> Each message in the partition has its offset number. Kinda like a length from the queue head.

1. Consumer consumes a message by specifying the offset of the **last** message they've consumed
2. Consumer periodically commits its offset to "Kafka" so that she can maintain what the latest offset is. The consumer has its current offset locally, but it also commits to Kafka for robustness. If consume fails and "forgets" its offset, he gets it from Kafka.

**Cluster** - ensures durability and availability. Involves robust replication mechanisms. 
- Each partition has a *leader* replica and one or more *follower* replica(s)
- Leader replica is responsible for all read and write requests from/to partition. The assignment of leader replica is managed centrally by cluster controller
- Follower replicas don't handle requests directly, they just passively replicate the data from the leader acting as a backup, ready to take over if the leader fails. They can resign on different brokers apart from leader replica. 