# kafka_connect_avro

Kafka REST schema registry
Kafka schema registry is not yet supported in Windows by Confluent. Also it is only available with Confluent kafka and not Apache Kafka.
Hence we need to run it through a Docker image.
docker-compose.yml
version: '2'

services:
  # this is our kafka cluster.
  kafka-cluster:
    image: landoop/fast-data-dev:cp3.3.0
    environment:
      ADV_HOST: 127.0.0.1         # Change to 192.168.99.100 if using Docker Toolbox
      RUNTESTS: 0                 # Disable Running tests so the cluster starts faster
      FORWARDLOGS: 0              # Disable running 5 file source connectors that bring application logs into Kafka topics
      SAMPLEDATA: 0               # Do not create sea_vessel_position_reports, nyc_yellow_taxi_trip_data, reddit_posts topics with sample Avro records.
    ports:
      - 2181:2181                 # Zookeeper
      - 3030:3030                 # Landoop UI
      - 8081-8083:8081-8083       # REST Proxy, Schema Registry, Kafka Connect ports
      - 9581-9585:9581-9585       # JMX Ports
      - 9092:9092                 # Kafka Broker
Docker command: docker-compose up
docker run -it --rm --net=host confluentinc/cp-schema-registry:3.3.1 bash
Then  run the kafka-avro producers and consumers with below commands:
# Produce a record with one field
kafka-avro-console-producer \
    --broker-list 127.0.0.1:9092 --topic test-avro \
    --property schema.registry.url=http://127.0.0.1:8081 \
    --property value.schema='{"type":"record","name":"myrecord","fields":[{"name":"f1","type":"string"}]}'

{"f1": "value1"}
{"f1": "value2"}
{"f1": "value3"}
# let's trigger an error:
{"f2": "value4"}
# let's trigger another error:
{"f1": 1}


# Consume the records from the beginning of the topic:
kafka-avro-console-consumer --topic test-avro \
    --bootstrap-server 127.0.0.1:9092 \
    --property schema.registry.url=http://127.0.0.1:8081 \
    --from-beginning


# Produce some errors with an incompatible schema (we changed to int) - should produce a 409
kafka-avro-console-producer \
    --broker-list localhost:9092 --topic test-avro \
    --property schema.registry.url=http://127.0.0.1:8081 \
    --property value.schema='{"type":"int"}'


# Some schema evolution (we add a field f2 as an int with a default)
kafka-avro-console-producer \
    --broker-list localhost:9092 --topic test-avro \
    --property schema.registry.url=http://127.0.0.1:8081 \
    --property value.schema='{"type":"record","name":"myrecord","fields":[{"name":"f1","type":"string"},{"name": "f2", "type": "int", "default": 0}]}'

{"f1": "evolution", "f2": 1 }

# Consume the records again from the beginning of the topic:
kafka-avro-console-consumer --topic test-avro \
    --bootstrap-server localhost:9092 \
    --from-beginning \
    --property schema.registry.url=http://127.0.0.1:8081
