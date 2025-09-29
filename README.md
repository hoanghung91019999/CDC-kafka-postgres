# CDC-kafka-postgres
## Apache kafka
- Apache Kafka là một nền tảng streaming data phân tán, open-source.

- Nó cho phép publish–subscribe message với tốc độ rất cao, đảm bảo độ bền dữ liệu và khả năng mở rộng.

- Kafka thường được dùng làm xương sống dữ liệu thời gian thực cho hệ thống lớn (banking, fintech, e-commerce, IoT…).

## Kiến trúc Kafka

- Kafka gồm 4 thành phần chính:

  + Producer

    + vd : Ứng dụng gửi dữ liệu vào Kafka (ví dụ: log, event, transaction).

  + Broker
    + Kafka server, lưu trữ và quản lý các message.
    + Một cluster Kafka thường có nhiều broker (3–5–7 node).

  + Topic & Partition
    + Topic = nơi chứa dữ liệu (giống như “table” trong database).
    + Partition = chia nhỏ topic để phân tán dữ liệu, tăng khả năng song song.
    + Message trong partition được lưu theo thứ tự (append-only log).

  + Consumer

    + Ứng dụng đọc dữ liệu từ Kafka.

    + Có thể đọc theo group (consumer group) để load balancing.

## Luồng dữ liệu Kafka
- Producer --> [Broker/Topic/Partition] --> Consumer

    + Producer gửi message → Kafka ghi vào log file trên disk (có replication giữa các broker).

    + Consumer đọc message theo offset (chỉ số vị trí).

    + Kafka không xóa message ngay sau khi consumer đọc → lưu theo retention policy (ví dụ 7 ngày, hoặc 1 TB).

## Thành phần mở rộng

- Ngoài core Kafka, có các tool mở rộng rất quan trọng:

    + ZooKeeper (trước Kafka 2.8)

      + Quản lý metadata, cluster state.

      + Kafka từ 2.8 trở lên có thể chạy KRaft mode (Kafka Raft) để bỏ dependency ZooKeeper.

    + Kafka Connect

      + Framework để kết nối Kafka với hệ thống khác (DB, S3, Elasticsearch, JDBC, Postgres…).

    + Kafka Streams

      + Library trong Java để xử lý stream trực tiếp từ Kafka (aggregate, join, window…).

    + MirrorMaker

      + Dùng để replicate dữ liệu giữa 2 Kafka cluster (cross-datacenter replication).

        + Ví dụ: Cluster A (on-premise) → Cluster B (cloud).

    + Schema Registry

      + Lưu trữ schema Avro/JSON/Protobuf.

        + Giúp producer và consumer thống nhất format dữ liệu.

## Các khái niệm quan trọng

- Offset: vị trí message trong partition.

- Replication Factor: số bản sao partition trên các broker (thường ≥3 để HA).

- Leader / Follower: mỗi partition có 1 leader, các broker khác giữ replica (follower).

- Retention: chính sách giữ message (theo thời gian hoặc dung lượng).

## Triển khai Kafka

- Có 3 cách phổ biến:

    + Manual (cài tay) → như bạn đang dùng (untar Kafka, chạy script).

    + Docker / Kubernetes → dễ scale, dễ quản lý.

    + Managed Service (AWS MSK, Confluent Cloud).

## Kịch bản thực tế

- Log & Metrics pipeline: App → Kafka → ELK/Clickhouse.

- Event-driven architecture: Microservices trao đổi qua Kafka.

- CDC (Change Data Capture): Postgres/MySQL → Kafka (Debezium) → Data Warehouse.

- Cross-datacenter replication: Dùng MirrorMaker.

- Streaming Analytics: Kafka → Flink/Spark → BI dashboard.

## Ưu điểm

- Throughput cực cao (hàng triệu msg/s).

- Bảo đảm thứ tự trong partition.

- Dữ liệu bền (ghi ra disk, replication).

- Scale out dễ dàng.

## Nhược điểm / Thách thức

- Quản lý cluster phức tạp (khi số node lớn).

- Tốn nhiều tài nguyên disk/IO.

- Phải có monitoring tốt (Prometheus, Grafana).

- Học cách tune parameter (retention, buffer, batch size…).
# Các Bước Deploy CDC kafka Postgres
## Chuẩn bị Postgres
- cấu hình postgres master
```
-- Bật wal_level = logical trong postgresql.conf
wal_level = logical
max_replication_slots = 4
max_wal_senders = 4
```
- tạo user replication
```
-- Tạo user replication
CREATE ROLE cdc_user WITH REPLICATION LOGIN PASSWORD 'cdc_pass';

-- Tạo publication cho bảng
CREATE PUBLICATION my_publication FOR TABLE my_table;
```
#### Cài Kafka và Zookeeper 
- Zookeeper – “bộ quản lý, điều phối”
  + Trong hệ sinh thái Kafka, Zookeeper (Apache ZooKeeper) đóng vai trò:
      + Quản lý cluster metadata: biết có bao nhiêu broker, broker nào đang sống/chết.
      + Quản lý leader election: khi 1 partition có nhiều replica, ZooKeeper quyết định broker nào là leader.
      + Lưu trữ thông tin cấu hình: topic, partition, ACL…
      + Theo dõi health check: broker nào mất kết nối thì cluster sẽ tự động bầu lại leader.
      + Zookeeper là “trưởng làng”, quản lý thông tin và điều phối để cluster Kafka hoạt động ổn định.
- Kafka – “máy phát tin nhắn”
  + Kafka broker là trái tim của hệ thống, nhiệm vụ chính:
      + Nhận dữ liệu từ producer (ứng dụng, connector, log collector…).
      + Ghi dữ liệu vào các topic → partition trên đĩa (theo dạng append-only log).
      + Phân phối dữ liệu cho consumer (các ứng dụng đọc/ETL, microservice, Spark, Flink…).
      + Đảm bảo tính bền vững và phân tán: dữ liệu được replicate trên nhiều broker.
      + Xử lý throughput cao: Kafka có thể xử lý hàng trăm nghìn đến hàng triệu messages/giây.
      + Kafka là “bưu điện tốc độ cao” lưu trữ và phân phối thông điệp.
- Mối quan hệ Kafka ↔ Zookeeper
  + Kafka broker không tự quản lý được metadata, leader election.
  + Kafka dựa vào ZooKeeper để:
      + Đăng ký khi broker start.
      + Nhận thông tin cluster.
      + Biết partition nào mình làm leader/follower.
      + Zookeeper = quản lý & điều phối
      + Kafka = lưu trữ & truyền dữ liệu
###### Cài kafka và zookerpeer phiên bản 3.8
```
wget https://downloads.apache.org/kafka/3.8.0/kafka_2.13-3.8.0.tgz
tar -xvzf kafka_2.13-3.8.0.tgz -C /opt/
ln -s /opt/kafka_2.13-3.8.0 /opt/kafka
```
- gói này bao gồm :
    + Kafka Broker
    + Kafka Connect, MirrorMaker, tools CLI
    + Zookeeper scripts (bin/zookeeper-server-start.sh)
- tạo kafka service systemd
```
[Unit]
Description=Apache Kafka Broker
After=zookeeper.service

[Service]
Type=simple
ExecStart=/opt/kafka/bin/kafka-server-start.sh /opt/kafka/config/server.properties
ExecStop=/opt/kafka/bin/kafka-server-stop.sh
Restart=on-abnormal

[Install]
WantedBy=multi-user.target
```
- tạo zookeeper.service systemd
```
[Unit]
Description=Apache Zookeeper
After=network.target

[Service]
Type=simple
ExecStart=/opt/kafka/bin/zookeeper-server-start.sh /opt/kafka/config/zookeeper.properties
ExecStop=/opt/kafka/bin/zookeeper-server-stop.sh
Restart=on-abnormal

[Install]
WantedBy=multi-user.target
```
- kích hoạt
```
systemctl daemon-reload
systemctl enable zookeeper kafka
systemctl start zookeeper kafka
```
- tạo kafka-connect systemd
```
[Unit]
Description=Kafka Connect Distributed Worker
After=kafka.service

[Service]
Type=simple
Environment="CONNECT_PLUGIN_PATH=/opt/debezium-plugins/debezium-connector-postgres,/opt/debezium-plugins/kafka-connect-jdbc/confluentinc-kafka-connect-jdbc-10.7.4"
ExecStart=/opt/kafka/bin/connect-distributed.sh /opt/kafka/config/connect-distributed.properties
Restart=always
User=root
WorkingDirectory=/opt/kafka

[Install]
WantedBy=multi-user.target

```
- cài debezium plugin
  + Debezium không phải là 1 service riêng, mà là bộ plugin (connector) chạy bên trong Kafka Connect.
  + Nó là “cầu nối” giúp Kafka Connect biết cách lấy dữ liệu từ database thông qua CDC (Change Data Capture).
  + Kafka Connect bản gốc chỉ là framework rỗng. Nếu không có plugin, nó không biết làm việc với PostgreSQL, MySQL, MongoDB, Oracle, ….
  + Với PostgreSQL → Debezium đọc WAL (Write Ahead Log) qua replication slot.
  + Với MySQL → Debezium đọc binlog
  + Chuyển đổi thay đổi (INSERT/UPDATE/DELETE) thành sự kiện JSON/Avro --> Đẩy sự kiện vào Kafka topic.
```
mkdir -p /opt/debezium-plugins/debezium-connector-postgres
cd /opt/debezium-plugins
tar -xvf /opt/debezium-plugins/debezium-postgres-2.7.4-plugin.tar.gz -C /opt/debezium-plugins/debezium-connector-postgres
ls -1 /opt/debezium-plugins/debezium-connector-postgres/*.jar | wc -l
systemctl restart kafka-connect
```
- sau khi cài xong debezium plugin sẽ trỏ đường dẫn kafka-connect vào pulgin
```
cd /opt/kafka/config
nano connect-distributed.properties
-- thêm đường dẫn plugin vừa cài
plugin.path=/opt/debezium-plugins/debezium-connector-postgres
```
- cài confluentinc-kafka-connect-jdbc-10.7.4
```
mkdir -p /opt/debezium-plugins/kafka-connect-jdbc
cd /opt/debezium-plugins/kafka-connect-jdbc
curl -L -o kafka-connect-jdbc-10.7.4.zip \
  https://api.hub.confluent.io/api/plugins/confluentinc/kafka-connect-jdbc/versions/10.7.4/confluentinc-kafka-connect-jdbc-10.7.4.zip
unzip kafka-connect-jdbc-10.7.4.zip -d /opt/debezium-plugins/kafka-connect-jdbc
-- Tải thêm JDBC driver cho PostgreSQL (bắt buộc)
wget https://jdbc.postgresql.org/download/postgresql-42.7.3.jar -P /opt/debezium-plugins/kafka-connect-jdbc
-- thêm đường dẫn plugin vừa cài
plugin.path=/opt/debezium-plugins/debezium-connector-postgres,/opt/debezium-plugins/kafka-connect-jdbc/confluentinc-kafka-connect-jdbc-10.7.4
systemctl restart kafka-connect
```
- Kiểm tra plugin đã load chưa
```
curl http://localhost:8083/connector-plugins
```
- tạo user trên db 2
```
sudo -u postgres psql
CREATE DATABASE db2;
CREATE USER db2_user WITH PASSWORD 'db2_pass';
GRANT ALL PRIVILEGES ON DATABASE db2 TO db2_user;
\c db2
GRANT ALL PRIVILEGES ON SCHEMA public TO db2_user;
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO db2_user;
GRANT ALL PRIVILEGES ON ALL SEQUENCES IN SCHEMA public TO db2_user;
```
- Tạo Source Connector (Debezium Postgres)
```
curl -X POST http://localhost:8083/connectors/ \
-H "Content-Type: application/json" -d '{
  "name": "source-db1",
  "config": {
    "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
    "database.hostname": "SERVER1_IP",
    "database.port": "5432",
    "database.user": "cdc_user",
    "database.password": "cdc_pass",
    "database.dbname": "db1",
    "database.server.name": "db1server",
    "plugin.name": "pgoutput",
    "publication.name": "my_publication",
    "slot.name": "my_slot"
  }
}'
```
- Tạo Sink Connector (JDBC → DB2)
```
curl -X POST http://localhost:8083/connectors/ \
-H "Content-Type: application/json" -d '{
  "name": "sink-db2",
  "config": {
    "connector.class": "io.confluent.connect.jdbc.JdbcSinkConnector",
    "connection.url": "jdbc:postgresql://SERVER3_IP:5432/db2",
    "connection.user": "db2_user",
    "connection.password": "db2_pass",
    "topics": "db1server.public.my_table",
    "auto.create": "true",
    "insert.mode": "upsert",
    "pk.mode": "record_key",
    "pk.fields": "id"
  }
}'
```
- Kiểm tra trạng thái connectors
```
curl http://localhost:8083/connectors/source-db1/status
curl http://localhost:8083/connectors/sink-db2/status
```
**nếu tất cả cùng running là thành công**
      
