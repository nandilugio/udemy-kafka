# Notes from presentation

Apache
    Kafka (core)
        PubSub messaging system
        Fast
            ZeroCopy
            Sequential disk writes to immutable Log
        Scalable
            Nodes are all equal
            Read/writes are load-balanced by sharding of Log Partitions in machines/disks
        Fault-tolerant
            Log Partitions can be replicated
            All nodes can take over
            Clients see all cluster through any node ??
        Durable
            Data is persisted in the Topic Logs
                Re-reading to recover from failures is easy
            Retention can be done by time, size or compaction (kep last timestamp for key/second)
        Flexible
            Exactly-once guarantees ??
            Ordering preserved at shard (Partition) level
            Many subscription paradigms
                Enables fault-tolerant data consumption and workload distribution
                - Queue (consumption within consumer group)
                - PubSub topics MOM-like (broadcasting between consumer group)s

Confluent
    Kafka Connect
    Kafka Streams
        Distributed Streaming Platflorm
    Kafka REST Proxy
    Schema Registry


Datastream decoupling.....

Topic/Feed
    Is a name
    Log: persistence method
        Immutable list of Records
        Consist of Partitions
            Scalability
            Replication of partitions for falut-tolerance
                Topic Replication factor = number of replicas per partition ??
                Kafka replication: failover
                Mirror Maker: Disaster recovery (e.g. mirroring whole cluster between AWS availability zones)

Partition
    Can be distributed (sharded) for parallel insertion and consumption of topic's records
    Can be replicated for parallel consumption and for fault-tolerance
        Leader: for writes
        Follower: catches up with changes on leader. If it's an ISR (in-sync replica) it can take leadership over
    Producer PICKS the partition
        Strategies
            Key
            Data
            Hash
            Priority
            Round robin
    Producer decides consistency level
        ack=0
        ack=1
        ack=all
    Offset pointers:
        Log end offset: end, where producers write to next
        High watermark: behind it, all records are all successfully replicated to all followers



Record
    Immutable

Producer

Consumer group
    Its a suscriber
    Has a unique ID
    Maintains its own offset pointer
        In `__consumer_offset` topic
    Record goes to one of the consumer of each group (Pub/Sub)
        Consumers in group load-balance consumption (each consume a fair-share of partitions)
        Only one consumer can read a particular partition
            If more partitions than consumer
                Some consumers read from more than one partition
            If more consumers than partitions, extra are idle
                Idle ones serve as failover
    Consumers ack processing (advancing the offset pointer)
        If the consumer dies before acking, another consumer gets the data
            This is at-least-once ?? #TODO now exactly-once no ??
                If so: Messages should be idempotent!
    Group coordinator
        A Broker handling addition/dissapearing of consumers in a group
            Redistributes shares automatically
    Consumers can see only records already replicated to all replicas (high-watermark)
    Reset policy #TODO




Cluster
    ZooKeeper
        Cluster coordination
    Broker: Node
        Has a numeric ID
        Contain Log Partitions

Sharding
    Partition decided normally by key

Hashing (`partitioner`)
  Default uses `murmur2` (see https://github.com/aappleby/smhasher):
    `targetPartition = Utils.abs(Utils.murmur2(record.key())) % numPartitions;`
      Same key => same partition
        Until `numPartition` changes!

# Kafka CLI

## Topics management

```sh
kafka-topics --zookeeper 127.0.0.1:2181
                                        --list
                                        --topic <name> --create --partitions 3 --replication-factor 1
                                                       --describe
                                                       --delete
```

## Console producer

If Kafka is configured to create new topics, production will 1st fail (producer issues a WARN), but then is retried until it succeeds, since the broker will have had the topic created.

```sh
kafka-console-producer --broker-list 127.0.0.1:9092 --topic <name>
                                                                   --producer-property acks=all
                                                                   --property parse.key=true --property key.separator=,
```

## Console consumer

```sh
kafka-console-consumer --bootstrap-server 127.0.0.1:9092 --topic <name>
                                                                        --from-beginning # Looks like from next-to-last commited instead of just new ones I think TODO
                                                                        --group <name>
                                                                        --property print.key=true --property key.separator=,
```

Q: if I consume to a point, commit, then consumer stops for a while, then consume _not from beginning_, then commit again, will I loose the messages in between?

## Consumer groups management

Consumer groups are automatically created for anonymous consumers. TODO: by console consumer? by broker?

```sh
kafka-consumer-groups --bootstrap-server 127.0.0.1:9092
                                                        --list
                                                        --group <name>
                                                                       --describe
                                                                       --topic <name> --reset-offsets (--to-earliest|--shift-by n|...) [--execute]
```

# 3rd party tools

## Alternatice CLIs

[edenhill/kafkacat](https://github.com/edenhill/kafkacat) See [Debugging with kafkacat](https://medium.com/@coderunner/debugging-with-kafkacat-df7851d21968)

## Kafka GUIs

[kafkatool](http://www.kafkatool.com/)

[yahoo/kafka-manager](https://github.com/yahoo/kafka-manager)


# Notable Config Options

## Producer Idempotence (Kafka >= 0.11)

```
# Producer
enable.idempotence=true
```

Adds an ID to deduplicate producer requests in the broker, in the case a broker retries because of a lost ack.

Also, it implies:

```
# Producer
acks=all
retries=Integer.MAX_VALUE
max.in.flight.requests= 1 (Kafka >= 0.11 & < 1.1)
                        5 (Kafka >= 1.1) still guarantees partition ordering!

```

Related to this, at the broker / topic level:

```
min.insync.replicas= n    Guarantees a min ISR
```

### Fine tuning:

```
acks= 0			Don't request acks from broker. Max throughput and max data loss risk.
      1			Request fastest ack from broker, ignoring replication state. Safeguards connection between producer and broker, but not between brokers in the cluster. TODO: acks _before_ writting to disk? makes throughput independent of disk io performance?
		  all   Request ack from brocker, from when at least `min.insync.replicas` replicas are in-sync.

retries= (int <= MAX_INT)  Retry sending messages. Me: scale to fit in memory when cluster isn't reachable.
         1                 No retries. Better throughput if data loss is admissible.  

max.in.flight.requests.per.connection= 1   Prevents ordering problems
																			 >1  In principle better throughput if partition ordering is not required. TODO: Kafka 1.1 defaults to 5 apparently still guaranteeing partition order. True? how?

```

## Message batching and buffering

Producers send messages as they arrive (initially one request per message to lower the latency), limiting inflight messages to `max.in.flight.requests.per.connection`. After the limit is reached, new messages are batched to be later sent all in one request, again to minimize latency and here also to increase throughput (less message overhead and better compression ratio).

In the case a producer receives messages faster than what the broker (TODO: partition?) can handle, the producer will:
  - Send `max.in.flight.requests.per.connection` requests, and stop sending until we get acks.
  - Queue the new messages in a buffer, but grouped in batches to be able to send them faster later since:
    - Less request overhead (less requests per message)
    - Better compression ratio (a bigger message has more repetition that can be deduplicated)
    Batches can be forced to be created even when in-flight requests limit is not exeeded, setting a `linger.ms` value > 0.
  - When the buffer is full (see `buffer.memory`), make `.send()` method blocking TODO: to induce backpressure?
  - If `.send()` needs to wait more than `max.block.ms`, an exception (TODO: which one?) is thrown in the producer. Meaning:
    - Producer's buffer is full
    - Broker is down or not accepting data
    - `max.block.ms`ms have elapsed since then

TODO: queues are per partition?

```
linger.ms= n      Allowed time to wait to build a batch. Adds to latency but yields better throughput (better compression, less requests). A full batch (see `batch.size`) will be sent immediatelly anyway.
batch.size= nKB   Limit to the size of a batch. Defaults to 16KB but 32 or 64 can be useful. Messages bigger than this will just not be batched.
buffer.memory=    Memory size of the buffer (request queue).
max.block.ms= n   Time `.send()` is allowed to block before throwing an exception.
```

TODO: monitor average batch size using Kafka Producer Metrics

## Message/batch compression

Set only at producer level. Broker stores message compressed. Consumers know they need to decompress without any config.

Better to compress, or compress more as:
  - Message is compressible (text)
  - Message or batch is big
  - Netwokrk is slow
  - Broker disk space is limited
  - Producer and consumer CPUs are able to handle the load

```
# Producer
compression.type= none      No compression
                  gzip      Most compression, but slower
                  lz4       Fast, but less compression
                  snappy    Fast, but less compression. See https://github.com/google/snappy
```

More info and benchmarks: https://blog.cloudflare.com/squeezing-the-firehose/




