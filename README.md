This project provides an automated and scalable Apache Kafka cluster management setup using Docker Compose. It leverages KRaft mode for a Zookeeper-less architecture and uses `bitnami/kafka` Docker images, tailored for local containerized environments and testing. Monitoring and management are provided via Prometheus, Grafana, a Kafka exporter, and Kafka UI.

# Kafka Docker Compose Setup (KRaft Mode with Bitnami Images)

This project provides a Docker Compose setup for running a Zookeeper-less Apache Kafka cluster locally using KRaft mode with `bitnami/kafka` images. It includes monitoring via Prometheus, Grafana, a Kafka exporter, and Kafka UI.

## Prerequisites

*   **Docker:** [Install Docker](https://docs.docker.com/get-docker/)
*   **Docker Compose:** [Install Docker Compose](https://docs.docker.com/compose/install/) (Usually included with Docker Desktop)

## KRaft-based Architecture Overview

This setup utilizes Kafka's KRaft (Kafka Raft Metadata) mode, which eliminates the need for a separate Zookeeper cluster. The cluster metadata is managed by a quorum of Kafka controllers running within the Kafka brokers themselves.

Key KRaft configuration parameters (found in `docker-compose.yml` under each Kafka service's environment section, prefixed with `KAFKA_CFG_`):
*   `KAFKA_CFG_PROCESS_ROLES`: Defines the roles of a node. In this setup, nodes are typically `broker,controller`.
*   `KAFKA_CFG_NODE_ID`: A unique ID for each Kafka node in the cluster.
*   `KAFKA_CFG_CLUSTER_ID`: A unique ID for the Kafka cluster. This is pre-generated for this setup.
*   `KAFKA_CFG_CONTROLLER_QUORUM_VOTERS`: A comma-separated list of `node_id@host:port` specifying the nodes that form the metadata quorum. Example: `1@kafka1:9091,2@kafka2:19094`.
*   `KAFKA_CFG_LISTENERS`: Defines the listeners the Kafka broker will bind to. This setup uses:
    *   `PLAINTEXT`: For inter-broker communication and internal Docker network clients (e.g., `PLAINTEXT://0.0.0.0:9092`).
    *   `CONTROLLER`: For KRaft metadata communication between controllers (e.g., `CONTROLLER://0.0.0.0:9091`).
    *   `EXTERNAL`: For clients connecting from outside the Docker network, i.e., from the host machine (e.g., `EXTERNAL://0.0.0.0:29092`).
*   `KAFKA_CFG_ADVERTISED_LISTENERS`: Comma-separated list of listeners with their resolvable hostnames/IPs that are advertised to clients.
*   `KAFKA_CFG_INTER_BROKER_LISTENER_NAME`: Specifies which listener name should be used for communication between brokers (e.g., `PLAINTEXT`).
*   `KAFKA_CFG_CONTROLLER_LISTENER_NAMES`: Specifies the listener name used by controllers (e.g., `CONTROLLER`).
*   `KAFKA_CFG_LISTENER_SECURITY_PROTOCOL_MAP`: Maps listener names to security protocols.
*   `KAFKA_CFG_OFFSETS_TOPIC_REPLICATION_FACTOR`: Replication factor for the consumer offsets topic.
*   `KAFKA_CFG_TRANSACTION_STATE_LOG_REPLICATION_FACTOR` & `KAFKA_CFG_TRANSACTION_STATE_LOG_MIN_ISR`: Replication factor and minimum in-sync replicas for the transaction state log, crucial in multi-broker KRaft setups.

## Getting Started

1.  **Clone the repository (or ensure `docker-compose.yml` and `prometheus.yml` are present in the current directory).**
    A `KAFKA_CFG_CLUSTER_ID` has been pre-generated in the `docker-compose.yml`. If you need to regenerate it for a completely new deployment, you can use a command like:
    ```bash
    docker run --rm bitnami/kafka:latest kafka-storage.sh random-uuid
    ```
    And then update `KAFKA_CFG_CLUSTER_ID` in `docker-compose.yml` for all brokers.

2.  **Start the Kafka cluster:**
    Open a terminal in the directory containing the `docker-compose.yml` file and run:
    ```bash
    docker-compose up -d
    ```
    This command will start the Kafka brokers along with the Kafka exporter, Prometheus, Grafana, and Kafka UI services in detached mode. Zookeeper is not used.

3.  **Verify the services are running:**
    You can check the status of the containers using:
    ```bash
    docker-compose ps
    ```
    You should see `kafka1`, `kafka2`, `kafka-exporter`, `prometheus`, `grafana`, and `kafka-ui` containers running.

## Accessing Kafka

The cluster now runs with two Kafka brokers, both acting as brokers and controllers:
*   **Broker 1 (`kafka1`):**
    *   External client access (from host): `localhost:29092` (maps to `EXTERNAL` listener)
    *   Internal Docker network (inter-broker, internal clients): `kafka1:9092` (maps to `PLAINTEXT` listener)
    *   Controller listener (internal): `kafka1:9091`
*   **Broker 2 (`kafka2`):**
    *   External client access (from host): `localhost:29093` (maps to `EXTERNAL` listener)
    *   Internal Docker network (inter-broker, internal clients): `kafka2:19092` (maps to `PLAINTEXT` listener)
    *   Controller listener (internal): `kafka2:19094`

**Bootstrap Servers for Kafka Clients:**
*   **From your host machine:** `localhost:29092,localhost:29093`
*   **From services within the same Docker network (`kafkanet`):** `kafka1:9092,kafka2:19092`

## Managing Topics and Partitions

### Automatic Topic Creation

By default, the Kafka brokers are configured with:
*   `KAFKA_CFG_AUTO_CREATE_TOPICS_ENABLE=true`.
*   `KAFKA_CFG_NUM_PARTITIONS=1` (default for auto-created topics).
*   `KAFKA_CFG_DEFAULT_REPLICATION_FACTOR=2` (default for auto-created topics, suitable for the 2-broker setup).

Modify these in `docker-compose.yml` for each broker and restart to change defaults.

### Manual Topic Management

Use Kafka command-line tools for more control.
1.  **Exec into a Kafka container** (e.g., `kafka1`):
    ```bash
    docker-compose exec kafka1 bash
    ```
2.  **Use `kafka-topics.sh`** (Bitnami images have tools in `/opt/bitnami/kafka/bin/` or `/opt/bitnami/kafka/bin/` often added to PATH, so `kafka-topics.sh` might be directly available).
    *   **Create a topic:**
        ```bash
        kafka-topics.sh --bootstrap-server localhost:9092 --create --topic my-kraft-topic --partitions 3 --replication-factor 2
        ```
        (Inside `kafka1`, `localhost:9092` is its `PLAINTEXT` listener. For `kafka2`, it would be `localhost:19092`.)
    *   **List topics:**
        ```bash
        kafka-topics.sh --bootstrap-server localhost:9092 --list
        ```
    *   **Describe a topic:**
        ```bash
        kafka-topics.sh --bootstrap-server localhost:9092 --describe --topic my-kraft-topic
        ```
3.  **Exit the container:** `exit`.

## Scaling and Configuration (KRaft Mode)

This setup runs two Kafka brokers (`kafka1`, `kafka2`), both acting as controllers and brokers.

**Key considerations for scaling KRaft:**
*   **Controller Quorum:** The `KAFKA_CFG_CONTROLLER_QUORUM_VOTERS` list is critical. It must be updated on all controller nodes when adding or removing controllers.
*   **Node IDs:** Each node (`KAFKA_CFG_NODE_ID`) must be unique.
*   **Listeners:** Ensure unique port mappings for external access and JMX for each new broker. Internal listener ports must also be unique if not using host networking.

**Adding More Brokers (e.g., `kafka3` as broker/controller):**
1.  **Open `docker-compose.yml`.**
2.  **Duplicate `kafka2`'s service block for `kafka3`.**
3.  **Modify for `kafka3`:**
    *   `container_name: kafka3`
    *   `KAFKA_CFG_NODE_ID: 3`
    *   Define new unique ports for `EXTERNAL` and `CONTROLLER` listeners, and map them in `ports`. Example:
        *   `ports`: `["29094:29094", "9997:9997"]` (External, JMX)
        *   `KAFKA_CFG_LISTENERS`: `PLAINTEXT://0.0.0.0:29095,CONTROLLER://0.0.0.0:19095,EXTERNAL://0.0.0.0:29094` (assuming `29095` for internal PLAINTEXT, `19095` for internal CONTROLLER)
        *   `KAFKA_CFG_ADVERTISED_LISTENERS`: `PLAINTEXT://kafka3:29095,EXTERNAL://localhost:29094`
    *   `KAFKA_JMX_PORT: 9997`
    *   Create a new volume: `kafka3_data:/bitnami/kafka` and define `kafka3_data: {}` under `volumes`.
4.  **Update `KAFKA_CFG_CONTROLLER_QUORUM_VOTERS` on ALL controller nodes** (`kafka1`, `kafka2`, and the new `kafka3`) to include the new controller. Example for 3 nodes:
    `KAFKA_CFG_CONTROLLER_QUORUM_VOTERS: '1@kafka1:9091,2@kafka2:19094,3@kafka3:19095'`
5.  **Adjust Replication Factors:** Update `KAFKA_CFG_OFFSETS_TOPIC_REPLICATION_FACTOR`, `KAFKA_CFG_TRANSACTION_STATE_LOG_REPLICATION_FACTOR`, `KAFKA_CFG_DEFAULT_REPLICATION_FACTOR` (and `KAFKA_CFG_TRANSACTION_STATE_LOG_MIN_ISR`) across all brokers to reflect the new cluster size (e.g., `3`).
6.  **Update `prometheus.yml`:** Add `kafka3:9997` to `targets` for the `kafka` job.
7.  **Update `docker-compose.yml`:** Add `kafka3` to `prometheus` service's `depends_on` list.
8.  Restart: `docker-compose up -d --force-recreate`. (Graceful rolling restarts are more complex and outside this scope).

## Monitoring

*   **Prometheus:** `http://localhost:9090`
    *   Targets: "Status" -> "Targets". Scrapes `kafka1:9999`, `kafka2:9998`, and `kafka-exporter:9308`.
*   **Kafka Exporter:** Exposes metrics on `http://localhost:9308` for Prometheus.
*   **Grafana:** `http://localhost:3000` (admin/admin)
    *   Add Prometheus data source: URL `http://prometheus:9090`.
    *   Import Kafka dashboards (e.g., from Grafana Labs, search for Bitnami Kafka or JMX ones).
*   **Kafka UI:** `http://localhost:8080`

## Data Persistence

Kafka message data and Grafana dashboards are persisted in Docker named volumes:
*   `kafka1_data` for `kafka1` (mounted at `/bitnami/kafka`)
*   `kafka2_data` for `kafka2` (mounted at `/bitnami/kafka`)
*   `grafana-storage` for Grafana (mounted at `/var/lib/grafana`)
These volumes are not deleted when you run `docker-compose down` unless you use the `-v` flag.

## Networking

All services are connected to a custom Docker bridge network named `kafkanet`, allowing them to resolve each other by service name (e.g., `kafka1`, `prometheus`).

## Stopping the Cluster

*   To stop and remove containers and the network:
    ```bash
    docker-compose down
    ```
*   To also remove named volumes (Kafka data, Grafana data if persisted):
    ```bash
    docker-compose down -v
    ```The `README.md` file has been comprehensively updated to reflect the transition to a KRaft-based Kafka setup using `bitnami/kafka` images.

**Key changes include:**

1.  **Introduction/Overview:** Updated to mention KRaft, Zookeeper-less architecture, and `bitnami/kafka` images.
2.  **Prerequisites:** Simplified, as Zookeeper is no longer a component.
3.  **KRaft-based Architecture Overview:** A new section was added explaining the core concepts of KRaft and detailing the key `KAFKA_CFG_` environment variables used in the `docker-compose.yml` for configuring KRaft mode, listeners, and replication.
4.  **Getting Started:**
    *   Instructions for regenerating `KAFKA_CFG_CLUSTER_ID` (if needed) were added.
    *   The list of services to verify running now correctly omits Zookeeper.
5.  **Accessing Kafka:**
    *   Updated to reflect the `EXTERNAL` (host access) and `PLAINTEXT` (internal Docker network) listeners for the Bitnami KRaft setup.
    *   Controller listener ports are also mentioned for clarity.
    *   Bootstrap server strings for both host and internal Docker network access are provided.
6.  **Managing Topics and Partitions:**
    *   Updated environment variable names to `KAFKA_CFG_...`.
    *   Clarified that `kafka-topics.sh` is available in Bitnami images (often in PATH).
    *   The example for creating topics now uses `my-kraft-topic`.
    *   Internal bootstrap server usage within `exec` is confirmed as still valid.
7.  **Scaling and Configuration (KRaft Mode):**
    *   This section was significantly rewritten to explain scaling in a KRaft context.
    *   Emphasized the critical role of `KAFKA_CFG_CONTROLLER_QUORUM_VOTERS` and the need to update it on all controller nodes.
    *   Provided a more detailed example for adding a `kafka3` node, including considerations for its controller listener port and updating replication factors.
8.  **Monitoring:**
    *   JMX scrape targets for Prometheus (`kafka1:9999`, `kafka2:9998`, and the `kafka-exporter` at `9308`) remain the same, but the surrounding text is consistent with the Bitnami setup.
9.  **Data Persistence:**
    *   A new section was added explaining that Kafka data is persisted in Docker named volumes (`kafka1_data`, `kafka2_data`) mounted at `/bitnami/kafka` (the default data directory for Bitnami Kafka images).
10. **Networking:**
    *   A new note was added about the `kafkanet` custom Docker network.
11. **General Cleanup:**
    *   All references to Zookeeper and Confluent-specific metric reporter variables were removed.
    *   Environment variables were updated to use the `KAFKA_CFG_` prefix as required by Bitnami images.
    *   Ensured overall consistency in naming, port conventions, and terminology.

The `README.md` should now accurately guide users in understanding, running, and managing this KRaft-based Kafka setup.
