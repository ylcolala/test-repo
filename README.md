version: '3.8'

services:

  kafka:
    image: confluentinc/cp-kafka:7.5.3
    container_name: kafka
    ports:
      - "9092:9092"
      - "9093:9093"

    environment:

      # KRaft mode
      KAFKA_PROCESS_ROLES: broker,controller
      KAFKA_NODE_ID: 1

      # Controller quorum
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@kafka:9093

      # Listeners
      KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9092,CONTROLLER://0.0.0.0:9093

      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092

      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT

      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER

      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT


      # Single node settings
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1

      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1

      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1


      # KRaft cluster id
      CLUSTER_ID: MkU3OEVBNTcwNTJENDM2Qk


    volumes:
      - kafka-data:/var/lib/kafka/data


  schema-registry:

    image: confluentinc/cp-schema-registry:7.5.3
    container_name: schema-registry

    depends_on:
      - kafka

    ports:
      - "8081:8081"


    environment:

      SCHEMA_REGISTRY_HOST_NAME: schema-registry

      SCHEMA_REGISTRY_LISTENERS: http://0.0.0.0:8081


      # Kafka bootstrap server
      SCHEMA_REGISTRY_KAFKASTORE_BOOTSTRAP_SERVERS: PLAINTEXT://kafka:9092


      # For KRaft, no ZooKeeper config needed


volumes:

  kafka-data: