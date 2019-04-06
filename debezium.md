[Debezium](https://debezium.io/) is an open source project for change data capture (CDC). It is built on Kafka Connect and supports multiple databases, such as MySQL, MongoDB, PostgreSQL, Oracle, and SQL Server. 
Pulsar provides a source connector over Kafka connect, so you can leverage Debezium to stream changes from your databases. 
This tutorial walks you through using pulsar source connector to store MySQL data changes. 

This tutorial is similar to [debezium tutorial](https://debezium.io/docs/tutorial), except that storage of event streams is changed from Kafka to Pulsar.
It mainly includes six steps: 
1. Start a MySQL server;
2. Start standalone Pulsar service;
3. Start Pulsar connector. Pulsar connector reads database changes existing in MySQL server;
4. Subscribe Pulsar topics to monitor MySQL changes;
5. Make changes in MySQL server, and verify that changes are recorded in Pulsar topics immediately;
6. Clean up.

## Step 1: Start a MySQL server
Start a MySQL server that contains a database example, from which Debezium captures changes. Open a new terminal to start a new container that runs a MySQL database server pre-configured with a database named inventory:
```
docker run --rm --name mysql -p 3306:3306 -e MYSQL_ROOT_PASSWORD=debezium -e MYSQL_USER=mysqluser -e MYSQL_PASSWORD=mysqlpw debezium/example-mysql:0.8
```

The following information is displayed:
```
2019-03-25T14:12:41.178325Z 0 [Note] Event Scheduler: Loaded 0 events
2019-03-25T14:12:41.178670Z 0 [Note] mysqld: ready for connections.
Version: '5.7.25-log'  socket: '/var/run/mysqld/mysqld.sock'  port: 3306  MySQL Community Server (GPL)
```

## Step 2: Start standalone Pulsar service
Start Pulsar service locally in standalone mode.
Debezium connector is introduced in Pulsar 2.3.0. 
Download [Pulsar binary of 2.3.0 release](https://www.apache.org/dyn/mirrors/mirrors.cgi?action=download&filename=pulsar/pulsar-2.3.0/apache-pulsar-2.3.0-bin.tar.gz) and [pulsar-io-kafka-connect-adaptor-2.3.0.nar of 2.3.0 release](https://www.apache.org/dyn/mirrors/mirrors.cgi?action=download&filename=pulsar/pulsar-2.3.0/connectors/pulsar-io-kafka-connect-adaptor-2.3.0.nar).

```
$ wget https://www.apache.org/dyn/mirrors/mirrors.cgi?action=download&filename=pulsar/pulsar-2.3.0/apache-pulsar-2.3.0-bin.tar.gz -O apache-pulsar-2.3.0-bin.tar.gz
$ wget https://www.apache.org/dyn/mirrors/mirrors.cgi?action=download&filename=pulsar/pulsar-2.3.0/connectors/pulsar-io-kafka-connect-adaptor-2.3.0.nar -O pulsar-io-kafka-connect-adaptor-2.3.0.nar
$ tar zxf apache-pulsar-2.3.0-bin.tar.gz
$ cd apache-pulsar-2.3.0
$ mkdir connectors
$ cp ../pulsar-io-kafka-connect-adaptor-2.3.0.nar connectors
$ bin/pulsar standalone
```

![picture-1](images/debezium-mysql-1.jpg)

## Step 3: Start Pulsar connector
Start a Pulsar debezium connector, with local run mode, in another terminal tab.
The “debezium-mysql-source-config.yaml” file contains all the configuration, and main parameters are listed under the “configs” node. The .yaml file contains the "task.class" parameter, and Pulsar connector is a wrapper for the parameter. The configuration file also 
includes MySQL related parameters (like server, port, user, password) and two names of Pulsar topics for "history" and "offset" storage.

```
$ bin/pulsar-admin source localrun  --sourceConfigFile debezium-mysql-source-config.yaml
``` 

The content in the “debezium-mysql-source-config.yaml” file is as follows.
```
tenant: "test"
namespace: "test-namespace"
name: "debezium-kafka-source"
topicName: "kafka-connect-topic"
archive: "connectors/pulsar-io-kafka-connect-adaptor-2.3.0.nar"

parallelism: 1

configs:
  ## sourceTask
  task.class: "io.debezium.connector.mysql.MySqlConnectorTask"

  ## config for mysql, docker image: debezium/example-mysql:0.8
  database.hostname: "localhost"
  database.port: "3306"
  database.user: "debezium"
  database.password: "dbz"
  database.server.id: "184054"
  database.server.name: "dbserver1"
  database.whitelist: "inventory"

  database.history: "org.apache.pulsar.io.debezium.PulsarDatabaseHistory"
  database.history.pulsar.topic: "history-topic"
  database.history.pulsar.service.url: "pulsar://127.0.0.1:6650"
  ## KEY_CONVERTER_CLASS_CONFIG, VALUE_CONVERTER_CLASS_CONFIG
  key.converter: "org.apache.kafka.connect.json.JsonConverter"
  value.converter: "org.apache.kafka.connect.json.JsonConverter"
  ## PULSAR_SERVICE_URL_CONFIG
  pulsar.service.url: "pulsar://127.0.0.1:6650"
  ## OFFSET_STORAGE_TOPIC_CONFIG
  offset.storage.topic: "offset-topic"

```

Tables are created automatically in the aforementioned MySQL server. So the Pulsar connector reads history records from MySQL binlog file from the beginning. In the output you will find the connector has already been processed within 47 records. 



For more information on how to manage connectors, see http://pulsar.apache.org/docs/en/io-managing/.

Records that have been captured and read by Debezium are automatically published to Pulsar topics. When you start a new terminal, you will find the current topics in Pulsar with the following command:
```
$ bin/pulsar-admin topics list public/default 
```


For each table, which has been changed, the change data is stored in a separate Pulsar topic.  Except database table related topics, another two topics named “history-topic” and “offset-topic” are used to store history and offset related data.
```
persistent://public/default/history-topic
persistent://public/default/offset-topic
```

## Step 4: Subscribe Pulsar topics to monitor MySQL changes
Take the `persistent://public/default/dbserver1.inventory.products` topic as an example.
Use the CLI command to consume this topic and monitor changes while the “products” table changes.
```
 $ bin/pulsar-client consume -s "sub-products" public/default/dbserver1.inventory.products -n 0
```

The output is as follows:
```
…
22:17:41.201 [pulsar-client-io-1-1] INFO  org.apache.pulsar.client.impl.ConsumerImpl - [public/default/dbserver1.inventory.products][sub-products] Subscribing to topic on cnx [id: 0xfe0b4feb, L:/127.0.0.1:55585 - R:localhost/127.0.0.1:6650]
22:17:41.223 [pulsar-client-io-1-1] INFO  org.apache.pulsar.client.impl.ConsumerImpl - [public/default/dbserver1.inventory.products][sub-products] Subscribed to topic on localhost/127.0.0.1:6650 -- consumer: 0
```

You can also consume the offset topic to monitor the offset changes while the table changes are stored in the `persistent://public/default/dbserver1.inventory.products` pulsar topic.
```
$ bin/pulsar-client consume -s "sub-offset" offset-topic -n 0   
```


## Step 5: Make changes in MySQL server, and verify that changes are recorded in Pulsar topics immediately

Start a MySQL CLI docker connector,  and you can make changes to the “products” table in MySQL server.
```
$docker run -it --rm --name mysqlterm --link mysql --rm mysql:5.7 sh -c 'exec mysql -h"$MYSQL_PORT_3306_TCP_ADDR" -P"$MYSQL_PORT_3306_TCP_PORT" -uroot -p"$MYSQL_ENV_MYSQL_ROOT_PASSWORD"'
```

After running the command, MySQL CLI is displayed, and you can change the names of the two items in the “products” table.

```
mysql> use inventory;
mysql> show tables;
mysql> SELECT * FROM  products ;
mysql> UPDATE products SET name='1111111111' WHERE id=101;
mysql> UPDATE products SET name='1111111111' WHERE id=107;
```



In the terminal where you consume products topic, you find that two changes have been added.



In the terminal where you consume the offset topic, you find that two offsets have been added.




In the terminal where you local-run the connector, you find two more records have been processed.



## Step 6: Clean up. 

Use “Ctrl +C” to close terminals. Use “docker ps” and “docker kill” to stop MySQL related docker instances. 
```
mysql> quit

$ docker ps
CONTAINER ID        IMAGE                        COMMAND                  CREATED             STATUS              PORTS                               NAMES
84d66c2f591d        debezium/example-mysql:0.8   "docker-entrypoint.s…"   About an hour ago   Up About an hour    0.0.0.0:3306->3306/tcp, 33060/tcp   mysql

$ docker kill 84d66c2f591d
```

To delete Pulsar data, delete data directory in the Pulsar binary directory.

```
$ pwd
/Users/jia/ws/releases/apache-pulsar-2.3.0

$ rm -rf data 
```


