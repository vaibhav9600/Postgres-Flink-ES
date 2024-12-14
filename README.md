# Flink SQL with PostgreSQL and Elasticsearch Integration

This README provides step-by-step instructions to set up a Flink SQL environment that integrates with PostgreSQL and Elasticsearch. Follow these steps to create a real-time pipeline for processing and visualizing data changes in Kibana.

## Prerequisites

1. Apache Flink 1.18.0
2. Docker and Docker Compose
3. PostgreSQL
4. Elasticsearch and Kibana

---

## Steps to Set Up and Run the Application

### 1. Start Flink Cluster

1. Open a terminal and navigate to the Flink directory (`flink-1.18.0`).
2. Run the following command to start the Flink cluster:

    ```bash
    ./bin/start-cluster.sh
    ```

3. Launch the Flink SQL client by running:

    ```bash
    ./bin/sql-client.sh
    ```

    This will start the Flink SQL terminal.

### 2. Start PostgreSQL

1. Open another terminal and bring up the PostgreSQL service with Docker Compose:

    ```bash
    docker-compose up -d
    ```

2. Access the PostgreSQL terminal by running:

    ```bash
    docker-compose exec postgres psql -h localhost -U postgres
    ```

3. Run the following SQL commands to create and initialize the database tables:

    ```sql
    CREATE TABLE shipments (
        shipment_id SERIAL NOT NULL PRIMARY KEY,
        order_id SERIAL NOT NULL,
        origin VARCHAR(255) NOT NULL,
        destination VARCHAR(255) NOT NULL,
        is_arrived BOOLEAN NOT NULL
    );
    ALTER SEQUENCE public.shipments_shipment_id_seq RESTART WITH 1001;
    ALTER TABLE public.shipments REPLICA IDENTITY FULL;
    INSERT INTO shipments
    VALUES (default, 10001, 'Beijing', 'Shanghai', false),
           (default, 10002, 'Hangzhou', 'Shanghai', false),
           (default, 10003, 'Shanghai', 'Hangzhou', false);

    CREATE TABLE orders (
        order_id INT PRIMARY KEY,
        customer_name VARCHAR(100),
        order_date TIMESTAMP
    );

    INSERT INTO orders VALUES
        (10001, 'Alice', '2024-12-01 10:00:00'),
        (10002, 'Bob', '2024-12-02 11:30:00'),
        (10003, 'Charlie', '2024-12-03 14:15:00');

    ALTER TABLE public.orders REPLICA IDENTITY FULL;
    ```

### 3. Configure Flink SQL

1. In the Flink SQL terminal, configure state checkpointing:

    ```sql
    SET execution.checkpointing.interval = 3s;
    ```

2. Create Flink tables for PostgreSQL CDC connectors:

    ```sql
    CREATE TABLE orders (
        order_id INT,
        customer_name STRING,
        order_date TIMESTAMP(3),
        PRIMARY KEY (order_id) NOT ENFORCED
    ) WITH (
        'connector' = 'postgres-cdc',
        'hostname' = 'localhost',
        'port' = '5434',
        'username' = 'postgres',
        'password' = 'postgres',
        'database-name' = 'postgres',
        'schema-name' = 'public',
        'table-name' = 'orders',
        'slot.name' = 'flink_orders'
    );

    CREATE TABLE shipments (
        shipment_id INT,
        order_id INT,
        origin STRING,
        destination STRING,
        is_arrived BOOLEAN,
        PRIMARY KEY (shipment_id) NOT ENFORCED
    ) WITH (
        'connector' = 'postgres-cdc',
        'hostname' = 'localhost',
        'port' = '5434',
        'username' = 'postgres',
        'password' = 'postgres',
        'database-name' = 'postgres',
        'schema-name' = 'public',
        'table-name' = 'shipments',
        'slot.name' = 'flink_shipments'
    );
    ```

3. Create a Flink table for Elasticsearch integration:

    ```sql
    CREATE TABLE orders_shipments_es (
        order_id INT,
        customer_name STRING,
        order_date TIMESTAMP(3),
        shipment_id INT,
        origin STRING,
        destination STRING,
        is_arrived BOOLEAN,
        PRIMARY KEY (order_id) NOT ENFORCED
    ) WITH (
        'connector' = 'elasticsearch-7',
        'hosts' = '',
        'username' = 'elastic',
        'password' = '',
        'index' = 'orders_shipments_v1'
    );
    ```

4. Insert data into Elasticsearch:

    ```sql
    INSERT INTO orders_shipments_es
    SELECT
        o.order_id,
        o.customer_name,
        o.order_date,
        s.shipment_id,
        s.origin,
        s.destination,
        s.is_arrived
    FROM
        orders AS o
    LEFT JOIN
        shipments AS s
    ON
        o.order_id = s.order_id;
    ```

### 4. Test Data Changes

1. Insert a new record into the `orders` and `shipments` tables in PostgreSQL:

    ```sql
    INSERT INTO orders
    VALUES (10004, 'Jark', '2020-07-30 15:22:00');

    INSERT INTO shipments
    VALUES (default, 10004, 'Shanghai', 'Beijing', false);
    ```

2. Update existing records in PostgreSQL:

    ```sql
    UPDATE orders SET customer_name = 'Jark Mo' WHERE order_id = 10004;

    UPDATE shipments SET is_arrived = true WHERE shipment_id = 1003;
    ```

3. Verify the changes reflected in Elasticsearch and visualize them in Kibana.

---

## Notes

- Ensure that PostgreSQL and Flink CDC connectors are properly configured.
- Adjust Elasticsearch connection details (e.g., `hosts`, `username`, and `password`) as per your setup.
- Data changes in PostgreSQL will automatically propagate to Elasticsearch due to the CDC pipeline.

## Conclusion

This setup demonstrates the integration of Flink, PostgreSQL, and Elasticsearch for real-time data streaming and analytics. You can extend this pipeline to include additional transformations or destinations as per your requirements.

