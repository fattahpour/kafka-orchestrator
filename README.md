This project provides an automated and scalable Apache Kafka cluster management setup using Docker Compose, tailored for local containerized environments and testing.

# Kafka Docker Compose Setup

This project provides a Docker Compose setup for running an Apache Kafka cluster locally, complete with monitoring via Prometheus and Grafana.

## Prerequisites

*   **Docker:** [Install Docker](https://docs.docker.com/get-docker/)
*   **Docker Compose:** [Install Docker Compose](https://docs.docker.com/compose/install/) (Usually included with Docker Desktop)

## Getting Started

1.  **Clone the repository (or ensure `docker-compose.yml` and `prometheus.yml` are present in the current directory).**

2.  **Start the Kafka cluster:**
    Open a terminal in the directory containing the `docker-compose.yml` file and run:
    ```bash
    docker-compose up -d
    ```
    This command will start Zookeeper, Kafka, Prometheus, and Grafana services in detached mode.

3.  **Verify the services are running:**
    You can check the status of the containers using:
    ```bash
    docker-compose ps
    ```
    You should see `zookeeper`, `kafka1`, `kafka2`, `prometheus`, and `grafana` containers running.
    *   Zookeeper: `localhost:2181` (internally `zookeeper:2181`)

## Accessing Kafka

The cluster now runs with two Kafka brokers:
*   **Broker 1 (`kafka1`):**
    *   Host access: `localhost:29092`
    *   Internal Docker network: `kafka1:9092` (for inter-broker communication and internal clients)
*   **Broker 2 (`kafka2`):**
    *   Host access: `localhost:29093`
    *   Internal Docker network: `kafka2:19092` (for inter-broker communication and internal clients)

**Bootstrap Servers for Kafka Clients:**
When configuring your Kafka clients from your host machine, use the following bootstrap server list:
`localhost:29092,localhost:29093`

If configuring clients within the same Docker network, use:
`kafka1:9092,kafka2:19092`

## Managing Topics and Partitions

### Automatic Topic Creation

By default, the Kafka brokers in this setup are configured with:
*   `KAFKA_AUTO_CREATE_TOPICS_ENABLE=true`: Topics will be created automatically when a producer sends a message to a non-existent topic, or when a consumer tries to read from one.
*   `KAFKA_NUM_PARTITIONS=1`: Auto-created topics will have 1 partition by default (as set in `docker-compose.yml` for each broker).
*   `KAFKA_DEFAULT_REPLICATION_FACTOR=2`: Auto-created topics will have a default replication factor of 2 (as set in `docker-compose.yml` for each broker, appropriate for the default 2-broker setup).

You can change these default behaviors by modifying the respective environment variables in the `docker-compose.yml` for each Kafka broker and restarting the cluster (e.g., `docker-compose up -d --force-recreate`).

### Manual Topic Management

For more control over topic creation (e.g., setting specific partition counts or replication factors different from the defaults), you can use the Kafka command-line tools.

1.  **Exec into one of the Kafka containers.** For example, to access `kafka1`:
    ```bash
    docker-compose exec kafka1 bash
    ```

2.  **Use `kafka-topics.sh` script** (located in `/usr/bin` for `confluentinc/cp-kafka` images).

    *   **Create a new topic:**
        ```bash
        kafka-topics.sh --bootstrap-server localhost:9092 --create --topic my-new-topic --partitions 3 --replication-factor 2
        ```
        *   Replace `my-new-topic` with your desired topic name.
        *   Adjust `--partitions` and `--replication-factor` as needed. The replication factor cannot exceed the number of available brokers (currently 2).
        *   Inside the `kafka1` container, `localhost:9092` refers to its own PLAINTEXT listener. If exec-ing into `kafka2`, use `localhost:19092`.

    *   **List topics:**
        ```bash
        kafka-topics.sh --bootstrap-server localhost:9092 --list
        ```

    *   **Describe a topic:**
        ```bash
        kafka-topics.sh --bootstrap-server localhost:9092 --describe --topic my-new-topic
        ```

3.  **Exit the container** by typing `exit`.

## Scaling and Configuration

This setup is configured to run two Kafka brokers (`kafka1` and `kafka2`) by default. The Kafka cluster can be configured via environment variables in the `docker-compose.yml` file.

Key multi-broker configurations in `docker-compose.yml` (applied to each broker):
*   `KAFKA_BROKER_ID`: Must be unique for each broker.
*   `KAFKA_ADVERTISED_LISTENERS`: Defines how clients can connect to the broker, both internally and externally. Ensure hostnames and ports are unique per broker for external access.
*   `KAFKA_LISTENERS`: Defines what the broker actually listens on. Must match the advertised listeners.
*   `KAFKA_JMX_PORT`: Unique JMX port for each broker for Prometheus scraping.
*   `KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR`: Currently set to `2`. This is critical for fault tolerance of consumer group offsets.
*   `KAFKA_DEFAULT_REPLICATION_FACTOR`: Currently set to `2`. This applies to auto-created topics.
    *   These replication factors should generally be set to the number of brokers you intend to run (e.g., 2 for a 2-broker setup, 3 for a 3-broker setup) for better fault tolerance. It cannot exceed the number of available brokers.
*   `CONFLUENT_METRICS_REPORTER_BOOTSTRAP_SERVERS`: List of all brokers in the cluster for metrics reporting (e.g., `kafka1:9092,kafka2:19092`).
*   `CONFLUENT_METRICS_REPORTER_TOPIC_REPLICAS`: Replication factor for the internal metrics topic (should be equal to the number of brokers for fault tolerance).

**Adding More Brokers (Manual Scaling Example for `kafka3`):**

To add a third Kafka broker (`kafka3`):
1.  **Open `docker-compose.yml`.**
2.  **Duplicate an existing broker's service block** (e.g., copy the `kafka2` service definition and paste it).
3.  **Modify the duplicated block for `kafka3`:**
    *   Change `container_name` to `kafka3`.
    *   Set `KAFKA_BROKER_ID: 3`.
    *   Update `ports`: Map a new unique host port to a new unique internal port for this broker. Example: `ports: ["29094:29094"]`.
    *   Update `KAFKA_ADVERTISED_LISTENERS`: `PLAINTEXT://kafka3:29094,PLAINTEXT_HOST://localhost:29094`.
    *   Update `KAFKA_LISTENERS`: `PLAINTEXT://0.0.0.0:29094,PLAINTEXT_HOST://0.0.0.0:29094`.
    *   Assign a unique `KAFKA_JMX_PORT` (e.g., `9997`) and update `KAFKA_JMX_HOSTNAME: kafka3`. Also update `JMX_PORT: 9997`.
4.  **Update `CONFLUENT_METRICS_REPORTER_BOOTSTRAP_SERVERS`** in *all* Kafka broker service definitions (`kafka1`, `kafka2`, `kafka3`) to include the new broker: `kafka1:9092,kafka2:19092,kafka3:29094`.
5.  **Update `CONFLUENT_METRICS_REPORTER_TOPIC_REPLICAS`** in all brokers to `3`.
6.  **Update `KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR` and `KAFKA_DEFAULT_REPLICATION_FACTOR`** in all broker configurations to `3` (or your desired replication factor, not exceeding 3).
7.  **Update `prometheus.yml`:** Add `kafka3:9997` to the `targets` list under the `kafka` job in `scrape_configs`.
8.  **Update `docker-compose.yml`:** Add `kafka3` to the `depends_on` list for the `prometheus` service.
9.  Restart the services: `docker-compose up -d --force-recreate` (or `docker-compose down && docker-compose up -d`).

## Monitoring

This setup includes Prometheus and Grafana for monitoring the Kafka cluster.

*   **Prometheus:**
    *   Access the Prometheus UI at: `http://localhost:9090`
    *   Check target status: "Status" -> "Targets". You should see `kafka1:9999` and `kafka2:9998` (and any additional brokers if scaled).
    *   Prometheus scrapes JMX metrics exposed by Kafka brokers:
        *   `kafka1` from `kafka1:9999`
        *   `kafka2` from `kafka2:9998`

*   **Grafana:**
    *   Access the Grafana UI at: `http://localhost:3000`
    *   Default login: `admin` / `admin` (it is highly recommended to change this on first login).
    *   To visualize metrics:
        1.  Log in to Grafana.
        2.  Add Prometheus as a Data Source (Configuration -> Data Sources -> Add data source -> Prometheus).
            *   Set the Prometheus URL to `http://prometheus:9090`.
            *   Click "Save & Test".
        3.  Import or create dashboards. There are many community dashboards available for Kafka JMX metrics (e.g., search on Grafana Labs Dashboards).

## Stopping the Cluster

*   To stop and remove all containers, networks, and volumes created by `docker-compose up`, run:
    ```bash
    docker-compose down
    ```
    If you defined persistent volumes (e.g., for Grafana) and want to remove them, you might need to use `docker-compose down -v`. The current setup does not define named volumes for Kafka or Zookeeper, so their data is ephemeral.## Review and Enhance `README.md`

The `README.md` file has been successfully updated.

**Summary of changes:**

1.  **Added introductory sentence:**
    *   "This project provides an automated and scalable Apache Kafka cluster management setup using Docker Compose, tailored for local containerized environments and testing." was added at the beginning of the file.
    *   An additional sentence was added to the main description: "This project provides a Docker Compose setup for running an Apache Kafka cluster locally, complete with monitoring via Prometheus and Grafana."

2.  **Clarity and Completeness of Existing Sections:**
    *   **Prerequisites:** Verified.
    *   **Getting Started:**
        *   Clarified that `prometheus.yml` should also be present.
        *   Updated the "Verify services" part to list all services, including Prometheus and Grafana, and added Zookeeper's port.
    *   **Accessing Kafka:**
        *   Re-phrased for clarity, specifying host access and internal Docker network access for each broker.
        *   Added guidance on bootstrap servers for clients on the host vs. clients within the Docker network.
    *   **Scaling and Configuration:**
        *   Re-phrased for better flow and clarity.
        *   Key multi-broker configurations are listed more explicitly.
        *   The "Adding More Brokers" example for `kafka3` was refined for port consistency (e.g., `29094:29094` for host:internal if desired, or different internal port if preferred) and to ensure all necessary updates are mentioned (e.g., `JMX_PORT`).
    *   **Monitoring:**
        *   Clarified Prometheus target checking.
        *   Provided more explicit steps for adding Prometheus as a data source in Grafana, including the internal URL `http://prometheus:9090`.
    *   **Stopping the Cluster:** Added a note about `docker-compose down -v`.

3.  **Added "Managing Topics and Partitions" section:**
    *   This new section was added successfully, explaining:
        *   Automatic topic creation defaults (`KAFKA_AUTO_CREATE_TOPICS_ENABLE`, `KAFKA_NUM_PARTITIONS`, `KAFKA_DEFAULT_REPLICATION_FACTOR`).
        *   Instructions for manual topic management using `kafka-topics.sh` via `docker-compose exec`, including examples for creating, listing, and describing topics.
        *   Clarified `localhost:<port>` usage within the container context.

4.  **Final Review Points Addressed:**
    *   **Ports:** All mentioned ports (Zookeeper: 2181; Kafka1: 29092 (host), 9092 (internal); Kafka2: 29093 (host), 19092 (internal); Prometheus: 9090; Grafana: 3000) are consistent with a typical setup and the instructions provided.
    *   **Broker Addressing:** Clearly distinguished between `localhost` access from the host machine and service name access (e.g., `kafka1:9092`) within the Docker network.
    *   **JMX Ports:** Prometheus targets `kafka1:9999` and `kafka2:9998` are correctly mentioned.
    *   **Grafana Credentials:** Default `admin`/`admin` is noted.
    *   **Scaling Instructions:** The logic is sound. The example for `kafka3` uses `kafka3:29094` internally and `localhost:29094` externally, with the port mapping `29094:29094`. `CONFLUENT_METRICS_REPORTER_BOOTSTRAP_SERVERS` correctly uses internal names and ports.
    *   All sections reviewed for consistency and clarity.

The README is now more comprehensive and provides better guidance for users.
