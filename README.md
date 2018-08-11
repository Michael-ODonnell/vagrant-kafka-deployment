# Vagrant Kafka Deployment
Simple Confluent.io open source Kafka deployment using vagrant and docker.

### Requirements
- [Vagrant](https://www.vagrantup.com)
- [Virtualbox](https://www.virtualbox.org/wiki/Downloads)

### Setup Environment
- Build Vagrant environment:
```sh
$ vagrant plugin install vagrant-docker-compose
$ vagrant up
```
- Synchronise local and remote folders:
```sh
$ vagrant rsync-auto
```
### Setup Database Within Influx Docker Container
#### TODO: Automate this
- In a new console window, still within the root of the repository, SSH into the virtual machine
```sh
$ vagrant ssh
```
- Start bash session within the influx docker container:
```sh
$ docker exec -it vagrant_influx_1 bash
```
- Enter the Influx Shell
```sh
$ influx \
    --host localhost \
    --port 8086
```
- Create a database and a user:
```sh
> CREATE DATABASE wrlddb
> CREATE USER root WITH PASSWORD 'root'
> GRANT ALL ON wrlddb to root
```

### Configure Kafka with an Influx Sink Connector
https://docs.confluent.io/current/connect/managing.html

#### TODO: Automate this
curl -X POST -H "Content-Type: application/json" --data '{"name": "InfluxTest", "config": {"connector.class":"com.datamountaineer.streamreactor.connect.influx.InfluxSinkConnector", "tasks.max":"1", "topics":"my-topic", "connect.influx.consistency.level":"ALL", "connect.influx.db":"wrlddb",  "connect.influx.error.policy":"THROW", "connect.influx.kcql":"INSERT INTO measureA SELECT * FROM my-topic", "connect.influx.max.retries":20,"connect.influx.username":"root", "connect.influx.password":"root", "connect.influx.retention.policy":"autogen", "connect.influx.retry.interval":60000, "connect.influx.url":"http://influx:8086","connect.progress.enabled":false }}' http://192.168.188.102:8083/connectors

### Send Mock Data to Kafka Instance
- In a new console window, still within the root of the repository, SSH into the virtual machine
```sh
$ vagrant ssh
```
- Start bash session within the kafka docker container:
```sh
$ docker exec -it schema_registry bash
```
- Navigate to `/usr/bin/`
```sh
$ cd /usr/bin/
```
- Create a `Kafka Avro console producer` to send mock data to the Kafka instance
```sh
$ ./kafka-avro-console-producer --broker-list broker-1:9092 --topic my-topic --property value.schema='{"type":"record","name":"measurement","fields":[{"name":"producer_guid","type":"string"},{"name":"type","type":"string"},{"name":"value","type":"double"},{"name":"userdata","type":"string"}]}'
```
- Pass in mock data (can be done multiple times):
```sh
{ "producer_guid":"65364685", "type":"thermometer", "value":23.48, "userdata":"{}"}
```
- If you wish to exit the current state, use the shortcut `Ctrl + C`

### View Mock Data in Kafka Instance
- In a new console window, still within the root of the repository, SSH into the virtual machine
```sh
$ vagrant ssh
```
- Start bash session within the kafka docker container:
```sh
$ docker exec -it vagrant_kafka_1 bash
```
- Navigate to `/usr/bin/`
```sh
$ cd /usr/bin/
```
- Create a `Kafka Avro console consumer` to accept mock data from the Kafka instance
```sh
$ ./kafka-avro-console-consumer \
    --bootstrap-server broker-1:9092 \
    --topic my-topic \
    --property value.schema='{"type":"record","name":"measurement","fields":[{"name":"producer_guid","type":"string"},{"name":"type","type":"string"},{"name":"value","type":"double"},{"name":"userdata","type":"string"}]}' \
    --from-beginning
```
- Observe data being passed to the console consumer
- If you wish to exit the current state, use the shortcut `Ctrl + C`

### View Mock Data in Influx Instance
- Return to the Influx Shell
 - Follow [earlier instructions](#setup-database-within-influx-docker-container) if the shell has been closed
- Select all records from `measureA` table
```sh
> USE wrlddb
> SHOW SERIES
> SELECT * FROM measureA
```
- Observe a record of data passed into the console producer
