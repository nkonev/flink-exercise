How-to guide: Build Streaming ETL for MySQL and Postgres based on Flink CDC (+Elasticsearch, Kibana)
[Old](https://www.ververica.com/blog/how-to-guide-build-streaming-etl-for-mysql-and-postgres-based-on-flink-cdc)
[New](https://ververica.github.io/flink-cdc-connectors/release-3.0/content/quickstart/mysql-postgres-tutorial.html)

1. Download Flink [1.18.0](https://archive.apache.org/dist/flink/flink-1.18.0/flink-1.18.0-bin-scala_2.12.tgz) and unpack
2. Download connectors
* https://mvnrepository.com/artifact/com.ververica/flink-sql-connector-mysql-cdc/3.0.0
* https://mvnrepository.com/artifact/com.ververica/flink-sql-connector-postgres-cdc/3.0.0
* https://repo.maven.apache.org/maven2/org/apache/flink/flink-sql-connector-elasticsearch7/3.0.1-1.17/flink-sql-connector-elasticsearch7-3.0.1-1.17.jar
and put them under `flink-1.18.0/lib/`
3. Start databases
```
docker-compose up -d
```

4. Preparing data in MySQL
```
# Check MySQL timezone of My SQL time by running one of the commands below
docker-compose exec mysql bash
mysql -e "SELECT @@global.time_zone;" -p123456
mysql -e "SELECT NOW();" -p123456

# set MySQL timezone
mysql -e "SET GLOBAL time_zone = '+4:00';" -p123456

Ctrl+D
```

Add data to MySQL
```
docker-compose exec mysql mysql -uroot -p123456

CREATE DATABASE mydb;
USE mydb;
CREATE TABLE products (
  id INTEGER NOT NULL AUTO_INCREMENT PRIMARY KEY,
  name VARCHAR(255) NOT NULL,
  description VARCHAR(512)
);
ALTER TABLE products AUTO_INCREMENT = 101;

INSERT INTO products
VALUES (default,"scooter","Small 2-wheel scooter"),
       (default,"car battery","12V car battery"),
       (default,"12-pack drill bits","12-pack of drill bits with sizes ranging from #40 to #3"),
       (default,"hammer","12oz carpenter's hammer"),
       (default,"hammer","14oz carpenter's hammer"),
       (default,"hammer","16oz carpenter's hammer"),
       (default,"rocks","box of assorted rocks"),
       (default,"jacket","water resistent black wind breaker"),
       (default,"spare tire","24 inch spare tire");

CREATE TABLE orders (
  order_id INTEGER NOT NULL AUTO_INCREMENT PRIMARY KEY,
  order_date DATETIME NOT NULL,
  customer_name VARCHAR(255) NOT NULL,
  price DECIMAL(10, 5) NOT NULL,
  product_id INTEGER NOT NULL,
  order_status BOOLEAN NOT NULL -- Whether order has been placed
) AUTO_INCREMENT = 10001;

INSERT INTO orders
VALUES (default, '2020-07-30 10:08:22', 'Jark', 50.50, 102, false),
       (default, '2020-07-30 10:11:09', 'Sally', 15.00, 105, false),
       (default, '2020-07-30 12:00:30', 'Edward', 25.25, 106, false);
```

5. Preparing data in Postgres
```
docker-compose exec postgres psql -U postgres

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
VALUES (default,10001,'Beijing','Shanghai',false),
       (default,10002,'Hangzhou','Shanghai',false),
       (default,10003,'Shanghai','Hangzhou',false);
```

6. Starting Flink cluster (actually JobManager and TaskManager)
```
./flink-1.18.0/bin/start-cluster.sh
```
Then we can visit http://localhost:8081/ to see if Flink is running normally, and the web page looks like

7. Start a Flink SQL CLI
```
./flink-1.18.0/bin/sql-client.sh
```

```
-- enable checkpoints every 3 seconds
SET execution.checkpointing.interval = 3s;

CREATE TABLE products (
    id INT,
    name STRING,
    description STRING,
    PRIMARY KEY (id) NOT ENFORCED
  ) WITH (
    'connector' = 'mysql-cdc',
    'hostname' = 'localhost',
    'port' = '3306',
    'username' = 'root',
    'password' = '123456',
    'database-name' = 'mydb',
    'table-name' = 'products'
  );
  
CREATE TABLE orders (
   order_id INT,
   order_date TIMESTAMP(0),
   customer_name STRING,
   price DECIMAL(10, 5),
   product_id INT,
   order_status BOOLEAN,
   PRIMARY KEY (order_id) NOT ENFORCED
 ) WITH (
   'connector' = 'mysql-cdc',
   'hostname' = 'localhost',
   'port' = '3306',
   'username' = 'root',
   'password' = '123456',
   'database-name' = 'mydb',
   'table-name' = 'orders'
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
   'port' = '5432',
   'username' = 'postgres',
   'password' = 'postgres',
   'database-name' = 'postgres',
   'schema-name' = 'public',
   'table-name' = 'shipments',
   'slot.name' = 'flink'
 );
 
-- create enriched_orders table that is used to load data to the Elasticsearch
CREATE TABLE enriched_orders (
   order_id INT,
   order_date TIMESTAMP(0),
   customer_name STRING,
   price DECIMAL(10, 5),
   product_id INT,
   order_status BOOLEAN,
   product_name STRING,
   product_description STRING,
   shipment_id INT,
   origin STRING,
   destination STRING,
   is_arrived BOOLEAN,
   PRIMARY KEY (order_id) NOT ENFORCED
 ) WITH (
     'connector' = 'elasticsearch-7',
     'hosts' = 'http://localhost:9200',
     'index' = 'enriched_orders'
 );
 
 
-- Enriching orders and load to ElasticSearch
INSERT INTO enriched_orders
 SELECT o.*, p.name, p.description, s.shipment_id, s.origin, s.destination, s.is_arrived
 FROM orders AS o
 LEFT JOIN products AS p ON o.product_id = p.id
 LEFT JOIN shipments AS s ON o.order_id = s.order_id;

```

8. Now, the enriched orders should be shown in Kibana. Visit http://localhost:5601/app/kibana#/management/kibana/index_pattern 
to create an index pattern `enriched_orders`.

Visit http://localhost:5601/app/kibana#/discover to find the enriched orders.

9. Play with it
Insert a new order in MySQL
```
--MySQL
INSERT INTO orders
VALUES (default, '2020-07-30 15:22:00', 'Jark', 29.71, 104, false);
```

Insert a shipment in Postgres
```
--PG
INSERT INTO shipments
VALUES (default,10004,'Shanghai','Beijing',false);
```

Update the order status in MySQL
```
--MySQL
UPDATE orders SET order_status = true WHERE order_id = 10004;
```

Update the shipment status in Postgres
```
--PG
UPDATE shipments SET is_arrived = true WHERE shipment_id = 1004;
```

Delete the order in MySQL
```
--MySQL
DELETE FROM orders WHERE order_id = 10004;
```
