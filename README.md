# Objectives
Create a step-by-step demo on how to stream tables from Postgres to Kafka/ksqlDB and back to Postgres.

## Data
- The data used here was originally taken from the [Graduate Admissions](https://www.kaggle.com/mohansacharya/graduate-admissions) open dataset available on Kaggle.
- The admit csv files are records of students and test scores with their chances of college admission.
- The research csv files contain a flag per student for whether or not they have research experience.

# Requirements
The following technologies are used through Docker containers:
* Kafka, the streaming platform
* Zookeeper, Kafka's best friend
* KSQL server, which we will use to create real-time updating tables
* Kafka's schema registry, needed to use the Avro data format
* Kafka Connect, pulled from [debezium](https://debezium.io/), which will source and sink data back and forth through Kafka
* Postgres, pulled from debezium, tailored for use with Connect

# How to Run

1. Build the image
```
docker build -t debezium-connect -f debezium.Dockerfile .
```
2. Activate the containers
```
docker-compose up -d
```
3. Login to PostgreSQL
```
docker run -it --rm --network=postgres-kafka-demo_default \
         -v $PWD:/home/data/ \
         postgres:11.0 psql -h postgres -U postgres
```
Password = postgres
Note: Dont forget to export your project directory path to PWD variable!
4. Create and connect to the database
```
CREATE DATABASE students;
```
```
\connect students;
```
5. Create table and insert data to table (admission)
```
CREATE TABLE admission (student_id INTEGER, gre INTEGER, toefl INTEGER, cpga DOUBLE PRECISION, admit_chance DOUBLE PRECISION, CONSTRAINT student_id_pk PRIMARY KEY (student_id));
```
```
\copy admission FROM '/home/data/admit_1.csv' DELIMITER ',' CSV HEADER
```
6. Create table and insert data to table (research)
```
CREATE TABLE research (student_id INTEGER, rating INTEGER, research INTEGER, PRIMARY KEY (student_id));
```
```
\copy research FROM '/home/data/research_1.csv' DELIMITER ',' CSV HEADER
```
7. Connect Postgres database to Kafka
```
curl -X POST -H "Accept:application/json" -H "Content-Type: application/json" \
      --data @postgres-source.json http://localhost:8083/connectors
```
8. See the list of existing connectors:
```
curl -H "Accept:application/json" localhost:8083/connectors/
```
9. Listing the available topics:
```
docker exec -it <cp-zookeeper-container-id> /bin/bash
```
```
/usr/bin/kafka-topics --list --zookeeper zookeeper:2181
```
10. Login to ksqlDB
```
docker run --network postgres-kafka-demo_default \
           --interactive --tty --rm \
           confluentinc/cp-ksql-cli:latest \
           http://ksql-server:8088
```
11. Configure some settings
```
set 'commit.interval.ms'='2000';
```
```
set 'cache.max.bytes.buffering'='10000000';
```
```
set 'auto.offset.reset'='earliest';
```
12. The Postgres table topics will be visible in ksqlDB, and we will create ksqlDB streams to auto update ksqlDB tables mirroring the Postgres tables:
```
SHOW TOPICS;
```
```
CREATE STREAM admission_src (student_id INTEGER, gre INTEGER, toefl INTEGER, cpga DOUBLE, admit_chance DOUBLE) WITH (KAFKA_TOPIC='dbserver1.public.admission', VALUE_FORMAT='AVRO');
```
```
CREATE STREAM admission_src_rekey WITH (PARTITIONS=1) AS SELECT * FROM admission_src PARTITION BY student_id;
```
```
SHOW STREAMS;
```
```
CREATE TABLE admission (student_id INTEGER, gre INTEGER, toefl INTEGER, cpga DOUBLE, admit_chance DOUBLE) WITH (KAFKA_TOPIC='ADMISSION_SRC_REKEY', VALUE_FORMAT='AVRO', KEY='student_id');
```
```
SHOW TABLES;
```
```
CREATE STREAM research_src (student_id INTEGER, rating INTEGER, research INTEGER) WITH (KAFKA_TOPIC='dbserver1.public.research', VALUE_FORMAT='AVRO');
```
```
CREATE STREAM research_src_rekey WITH (PARTITIONS=1) AS SELECT * FROM research_src PARTITION BY student_id;
```
```
CREATE TABLE research (student_id INTEGER, rating INTEGER, research INTEGER) WITH (KAFKA_TOPIC='RESEARCH_SRC_REKEY', VALUE_FORMAT='AVRO', KEY='student_id');
```
13. Create downstream tables. We will create a new ksqlDB streaming table to join students' chance of admission with research experience.
```
CREATE TABLE research_boost AS \
  SELECT a.student_id as student_id, \
         a.admit_chance as admit_chance, \
         r.research as research \
  FROM admission a \
  LEFT JOIN research r on a.student_id = r.student_id;
```

and another table calculating the average chance of admission for students with and without research experience:

```
CREATE TABLE research_ave_boost AS \
     SELECT research, SUM(admit_chance)/COUNT(admit_chance) as ave_chance \
     FROM research_boost \
     WITH (KAFKA_TOPIC='research_ave_boost', VALUE_FORMAT='delimited', KEY='research') \
     GROUP BY research;
```

14. Add a connector to sink a KSQL table back to Postgres

The postgres-sink.json configuration file will create a RESEARCH_AVE_BOOST
table and send the data back to Postgres.

```
curl -X POST -H "Accept:application/json" -H "Content-Type: application/json" \
      --data @postgres-sink.json http://localhost:8083/connectors
```

15. Update the source Postgres tables and watch the Postgres sink table update

The RESEARCH_AVE_BOOST table should now be available in Postgres to query:

```
SELECT "AVE_CHANCE" FROM "RESEARCH_AVE_BOOST" WHERE cast("RESEARCH" as INT)=0;
```

With these data the average admission chance will be 65.19%.

16. Add some new data to the admission and research tables in Postgres:

```
\copy admission FROM '/home/data/admit_2.csv' DELIMITER ',' CSV HEADER
```
```
\copy research FROM '/home/data/research_2.csv' DELIMITER ',' CSV HEADER
```

With the same query above on the RESEARCH_AVE_BOOST table, the average chance of admission for students without research experience has been updated to 63.49%.
