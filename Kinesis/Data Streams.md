Amazon Kinesis Data Streams (KDS) is a real-time, scalable data streaming service designed for continuous ingestion and processing of large volumes of data.
<img width="1100" height="441" alt="image" src="https://github.com/user-attachments/assets/323dbe54-9814-4e25-b6bb-3bdff9183691" />

Key concepts include:
### Data Stream:
A continuous flow of data records. It represents a logical grouping of data that can be processed by one or more consumers.
### Shard:
The fundamental unit of throughput and scale within a Kinesis Data Stream. Each shard provides a fixed capacity for both writes (1 MB/s and 1,000 records/second) and reads (2 MB/s). Data records within a shard are strictly ordered.
### Data Record:
The unit of data stored in a Kinesis Data Stream. Each record consists of a partition key, a sequence number, and a data blob.
### Partition Key:
A string used to group data records within a stream. KDS uses the partition key to determine which shard a data record is assigned to, ensuring that records with the same partition key are sent to the same shard and maintain order within that shard.
### Sequence Number:
A unique identifier for each data record within a shard. It reflects the order in which records are added to the shard.
### Producer:
An application or source that sends data records to a Kinesis Data Stream. Examples include web applications, IoT devices, or log aggregators.
### Consumer:
An application or service that reads and processes data records from a Kinesis Data Stream. Consumers can be built using the Kinesis Client Library (KCL), AWS Lambda, or other services like Apache Flink.
### Retention Period:
The duration for which data records are stored in a Kinesis Data Stream. The default is 24 hours, but it can be extended up to 7 days or even longer with extended retention.
### Capacity Modes:
KDS offers two capacity modes:
#### Provisioned: 
Users explicitly define the number of shards required, providing predictable throughput.
#### On-Demand: 
Kinesis automatically manages the scaling of shards based on data volume and throughput, simplifying capacity planning.
### Enhanced Fan-Out:
A feature that provides dedicated read throughput for each consumer application, preventing read congestion and ensuring consistent low latency for multiple consumers reading from the same shard.
Scaling:
KDS supports dynamic scaling by allowing users to increase or decrease the number of shards in a stream (shard splitting or merging) to adjust to changing data volumes.
<img width="720" height="473" alt="image" src="https://github.com/user-attachments/assets/2f8e41ed-840a-4eeb-aa6a-e21b9159bc2d" />

<img width="720" height="462" alt="image" src="https://github.com/user-attachments/assets/13e1bdce-4b4a-4f98-a120-ffeaeeeabbfe" />
