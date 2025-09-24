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
