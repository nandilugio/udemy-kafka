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


