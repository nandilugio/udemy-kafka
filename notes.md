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


