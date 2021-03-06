# Project goal

This project is meant to experiment with [Debezium MySQL CDC Kafka Connector](https://debezium.io/docs/connectors/mysql), to capture changes from a MySQL Database.

This includes all is required locally with Docker Compose, including:
- A 3 nodes Kafka cluster + ZooKeeper
- Schema Registry
- Kafka Connect + Debezium connector
- A MySQL instance with a sample database

**THIS IS STILL A WIP, far from being completed**

## Running the cluster 

[To run the cluster...](./docker/README.md)

Note that, due to the way Docker network is working, each broker is listening on two listeners, internally and externally of the docker network, and they have to use different ports:
1. LISTENER_DOCKER_INTERNAL://kafka1:19092
2. LISTENER_DOCKER_EXTERNAL://${DOCKER_HOST_IP:-127.0.0.1}:9092

Everything living inside the docker compose network, like the Kafka Connect instance, should use the internal port (19092...), while to connect to the cluster from an external application you must use the external port (9092...)

## Useful snippets

### Setting up the Connector

`POST`, `http://localhost:8083/connectors`

```
{
  "name": "inventory-connector",
  "config": {
    "connector.class": "io.debezium.connector.mysql.MySqlConnector",
    "database.hostname": "db",
    "database.port": "3306",
    "database.user": "debezium",
    "database.password": "dbz",
    "database.server.id": "12345",
    "database.server.name": "example",
    "database.whitelist": "inventory",
    "table.whitelist": "inventory.products,inventory.products_on_hand,inventory.customers,inventory.addresses,inventory.orders",
    "database.history.kafka.bootstrap.servers": "kafka1:19092",
    "database.history.kafka.topic": "dbhistory.inventory",
    "include.schema.changes": "true"
  }
}
```

Note that the Kafka Connect nodes is inside the Docker network so you must use the internal broker listener (19092).

To see the configuration and status of Connectors:

- List Connectors: `http://localhost:8083/connectors`
- View `inventory-connector` configuration: `http://localhost:8083/connectors/inventory-connector`
- View `inventory-connector` status: `http://localhost:8083/connectors/inventory-connector/status`


### Topic creations

Note that Debezium connector does NOT create the required topics explicitly.
The Kafka cluster in this example is set up with `auto.create.topic.enable` = `true`

If Topic auto-create is disable, you should create all required topics by hand:
(This requires the Confluent platform installed locally)

```
$ kafka-topics.sh --create --zookeeper localhost:2181 example
$ kafka-topics.sh --create --zookeeper localhost:2181 example.inventory.addresses
$ kafka-topics.sh --create --zookeeper localhost:2181 example.inventory.customers
$ kafka-topics.sh --create --zookeeper localhost:2181 example.inventory.orders
$ kafka-topics.sh --create --zookeeper localhost:2181 example.inventory.products
$ kafka-topics.sh --create --zookeeper localhost:2181 example.inventory.products_on_hand
```

You need:
- one topic for the server: `example` (`database.server.name` connector property)
- one topic per table, following the pattern `<server.name>.<database.name>.<table.name>`


### Watching the changelog topics

(This requires the Confluent platform installed locally)
```
$ kafka-avro-console-consumer --bootstrap-server localhost:9092 --property schema.registry.url=http://localhost:8081 --from-beginning --topic example.inventory.products | jq '.'
```

## Useful docs

### Kafka Connect config example
https://docs.confluent.io/current/installation/docker/docs/installation/connect-avro-jdbc.html

### Kafka Connect API
https://docs.confluent.io/current/connect/references/restapi.html

### Debezium connector configuration
https://debezium.io/docs/connectors/mysql/#deploying-a-connector


## Interestin features to add to the cluster

Add KConnect UI and Topics UI from Landoop
https://github.com/simplesteph/kafka-stack-docker-compose/blob/master/full-stack.yml
