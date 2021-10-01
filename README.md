# rabbit-load-generator

Tool to generate queue and message load that mimics CareAware's 3 main queue types:

1. quorum queues
1. classic durable mirrored queues
1. classic transient auto-delete queues

The following server-side policies are in place for each queue type:

![Queue Policies](policies.PNG)

## Usage With Docker

```shell
docker run -it --rm --network host pivotalrabbitmq/rabbit-load-generator \
  --spring.rabbitmq.addresses=amqp://guest:guest@localhost:5672// \
  --spring.profiles.active=qq
```

The available profiles are `qq`, `durable-mirrored`, `autodelete`.

It is possible to specify a list of addresses, separated by commas.
Use `/` in the URI to specify the `/`, not `%2F`.

The profile scenarios are configured in [application.yaml](src/main/resources/application.yml).
Options can be set on the command line, e.g:

```shell
docker run -it --rm --network host pivotalrabbitmq/rabbit-load-generator \
  --spring.rabbitmq.addresses=amqp://guest:guest@localhost:5672// \
  --spring.profiles.active=qq \
  --rabbit-load-generator.scenarios[0].connections=20
```

## Configuration

See the following classes for additional configuration:

1. [RabbitProperties](https://docs.spring.io/spring-boot/docs/current/api/org/springframework/boot/autoconfigure/amqp/RabbitProperties.html)
2. [RabbitLoadGeneratorProperties](src/main/java/com/cerner/test/RabbitLoadGeneratorProperties.java)

### NonProd-01

To mimic NONPROD-01 cluster, use the following configuration:

```yml
---
spring:
  profiles: 1k-qq
rabbit-load-generator:
  rabbitServiceName: rabbitmq-04
  scenarios:
  # Creates 33 connections, 330 channels, 990 quorum queues, 990 bindings, and 19800 consumers
  # For each of the 999 bindings, a message is published every 120 seconds for a total throughput of 8.3 msgs/sec
  # Start 4 instances of this profile to match CareAware's NONPROD-01 cluster
  - queueNamePrefix: quorum-
    uniqueExchange: true
    connections: 33
    channelsPerConnection: 10
    queuesPerChannel: 3
    consumersPerQueue: 20
    bindingsPerQueue: 1
    autoDelete: false
    durable: true
    quorum: true
    publishInterval: 120000
    publishPersistent: true
    publishMsgSizeBytes: 20000

---
spring:
  profiles: 1k-durable-mirrored
rabbit-load-generator:
  rabbitServiceName: rabbitmq-04
  scenarios:
  # Creates 33 connections, 330 channels, 990 durable classic queues, 990 bindings, and 9900 consumers
  # For each of the 999 bindings, a request/reply is performed every 60 seconds for a total throughput of 16.5 requests/sec
  # Start 7 instances of this profile to match CareAware's NONPROD-01 cluster
  - queueNamePrefix: classic-durable-mirrored-
    uniqueExchange: true
    connections: 33
    channelsPerConnection: 10
    queuesPerChannel: 3
    consumersPerQueue: 10
    bindingsPerQueue: 1
    autoDelete: false
    durable: true
    quorum: false
    publishInterval: 0
    requestInterval: 60000
    requestMsgSizeBytes: 10000
    replyMsgSizeBytes: 500000

---
spring:
  profiles: 1k-autodelete
rabbit-load-generator:
  rabbitServiceName: rabbitmq-04
  scenarios:
  # Creates 33 connections, 330 channels, 990 transient classic queues, 1980 bindings, and 990 consumers
  # For each of the 1998 bindings, a message is published every 240 seconds for a total throughput of 8.3 msgs/sec
  # Start 7 instances of this profile to match CareAware's NONPROD-01 cluster
  - queueNamePrefix: classic-autodelete-
    uniqueExchange: true
    connections: 33
    channelsPerConnection: 10
    queuesPerChannel: 3
    consumersPerQueue: 1
    bindingsPerQueue: 2
    autoDelete: true
    durable: false
    quorum: false
    publishInterval: 240000
    publishPersistent: false
    publishMsgSizeBytes: 20000
```

Running 4 instances of "1k-qq", 7 instances of "1k-durable-mirrored", and 7 instances of "1k-autodelete" will generate total cluster counts of:

1. 17820 queues
1. 594 connections
1. 5940 channels
1. 24750 bindings
1. 155430 consumers
1. 169 messages/sec