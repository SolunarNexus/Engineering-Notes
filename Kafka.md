> From high-level it is an event streaming platform used either as a message queue or a stream processing system.

**Producer** - entity responsible for publishing events in the queue (server, process, ...)

**Consumer** - entity taking event from the queue and changing its behavior (website, ...)

**Consumer group** - group of consumers where it is guaranteed that an event is processed by exactly one consumer from the group.

**Broker** - the servers (physical or virtual) that hold the event partitions (queues). Each can have multiple partitions.

**Partition** - the queue. An *ordered*, *immutable* sequence of messages that we append to (like a log file). Messages usually belong to the same logical origin (e.g. a particular football game). It's a physical grouping of messages - a way of scaling data.

**Topic** - a logical grouping of partitions - a way of organizing data (e.g. topic - football and partitions - football games or topic - basketball and partitions - basketball games). Producer publishes to, and consumers consumes from topic.

![[Pasted image 20260824202743.png]]

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
![[Pasted image 20260824202712.png]]

**Cluster** - ensures durability and availability. Involves robust replication mechanisms. 
- May consist of multiple brokers/servers
- Each partition has a *leader* replica and one or more *follower* replica(s)
- Leader replica is responsible for all read and write requests from/to partition. The assignment of leader replica is managed centrally by cluster controller
- Follower replicas don't handle requests directly, they just passively replicate the data from the leader acting as a backup, ready to take over if the leader fails. They can resign on different brokers apart from leader replica. 
![[Pasted image 20260824202622.png]]

**When to use Kafka**
1. *Whenever we need a message queue*: 
	- need to processs data async ie. YT transcoding
	- need for in-order message processing 
	- need to decouple producer and consumer so they can scale independently
2. *Whenever we need a stream*:
	- need to process a lot of data in real-time
	- need to process a stream of messages by multiple consumers simultaneously

**Scalability** - there are some constraints that helps to scale well:
- There is no hard limit for message size but it's recommended to not exceed **1MB per message**. Otherwise you might overwhelm network or memory. Don't send multimedia blob as a value of message, instead send an url to that multimedia (e.g. S3 url which the consumer takes and downloads the multimedia) 
- One broker should have maximum **1TB** of data and handle **10k** messages per second. It depends on the hardware and size of the message, but gives a good baseline for estimates when designing

**How to scale?**
1. *More brokers* - like a horizontal scaling, more memory, more processing power, more messages to store & handle
2. *Choose good partition key* -  good key evenly distributes messages across partition space
3. *Use managed service* - third-party option where a service does the scaling for you (e.g. AWS MSK)

**Hot partition** - (considering a good partition key) a lot of messages are assigned to a single partition that is being overwhelmend (e.g. popular event, person, etc.). To solve this:
1. *Remove the key* - it's possible, then the Kafka randomly distributes messages in partition space (e.g. using round-robin algorithm). It's fine if you don't care about continuous message ordering.
2. *Compound key* - append a (random) value to the partition key (e.g. random number in range 1-N where N is number of partitions, userId, etc.). This as well destroys the ordering.
3. *Backpressure* - slow down the producer

**Fault tolerance & durability** - there are some settings:
- *acks* - tells how many follower partitions need to acknowledge that they got the message before we can move on to the next message. *acks=all* is maximum durability. This setting affects the performance so we need to find a meaningful balance
- *replication factor* - how many followers we're gonna have (default is 3)
Good to mention that **Kafka doesn't go down** ever or hardly ever. That's because of durability guarantees of Kafka (cluster, cluster controller, multiple brokers, settings ...)

**What if consumer goes down?** - After consumer reads a message, it commits the offset (from message) to Kafka. If consumer goes down, when he is back up, he gets the offset from Kafka which remembers and consumer can continue where he left off.
> **Important**: it's a good practice to commit the offset **after** the work on consumer's end is done. If the offset is commited first thing after reading the message and the consumer goes down, after he gets back up it will think the work has been done successfully and kafka gives him the offset as it is true.   

**What if consumer group goes down?** - in such case rebalance the group. Each consumer in a group is responsible for a range of partitions. If one consumer goes down, we need to rebalance the group so that ranges are updated. That is handled by cluster.


**Errors & retries**
- *producer retries* - there's an API for enabling retry mechanisms e.g. exponential wait time between retries, initial wait time etc. It's important to enable idempotency to avoid duplicate messages when retrying to publish a message
- *consumer retries* - isn't supported by kafka out-of-the-box. If consumer fails to fetch a page from url in message, it can place that message on *retry topic* after first N failures. After MN failures the message can be placed in *DLQ topic* (dead letter queue). 

**Performance optimizations**
- *Batch messages in producer* - so that we send request to publish less often
- *Compress messages in producer* - saves space and throughput
- **Partition key - achieve even distribution over partition space**

**Retention policy** - 2 settings
- *retention.ms* - how long to keep a message (default 7 days)
- *retention.bytes* - how big the log file needs to be before we start to purge messages (default 1GB)
