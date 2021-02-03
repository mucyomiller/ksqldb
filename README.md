# ksqlDB command

## start ksqlDB instance
```shell
$  docker exec -it ksqldb-cli ksql http://ksqldb-server:8088
```

Set offset to earliest

```shell
ksql> SET 'auto.offset.reset' = 'earliest';
```

Create stream

```shell
ksql> CREATE STREAM MOVEMENTS (LOCATION VARCHAR) WITH (VALUE_FORMAT='JSON', PARTITIONS=1, KAFKA_TOPIC='movements');
```

Show streams

```shell
ksql> show  streams;
```

Show topics

```shell
ksql> show topics;
```

Clear console

```shell
ksql> clear
````

Select some data

```shell
ksql> SELECT TIMESTAMPTOSTRING(ROWTIME,'yyyy-MM-dd','Europe/Amsterdam') AS EVENT_TS, ROWKEY AS PERSON,LOCATION FROM MOVEMENTS EMIT CHANGES;
```
output:

| EVENTS_TS | PERSON | LOCATION |
| --- | --- | --- |
| 2020-02-26 19:23:24 | robin | York |
-----   


Insert data directly to stream

```shell
ksql> INSERT INTO MOVEMENTS VALUES ('robin', 'York');
```

Select where

```shell
ksql> SELECT TIMESTAMPTOSTRING(ROWTIME,'yyyy-MM-dd','Europe/Amsterdam') AS EVENT_TS, ROWKEY AS PERSON,LOCATION FROM MOVEMENTS WHERE LCASE(LOCATION)= 'york' EMIT CHANGES;
```

Stream it back to Kafka

```shell
ksql> CREATE STREAM FOO AS SELECT TIMESTAMPTOSTRING(ROWTIME,'yyyy-MM-dd','Europe/Amsterdam') AS EVENT_TS, ROWKEY AS PERSON,LOCATION FROM MOVEMENTS WHERE LCASE(LOCATION)= 'york' EMIT CHANGES;
```
show streams can now show you FOO also.

Show Queries

```shell
ksql> SHOW QUERIES
```

Describe Query   
(message-per-sec,total-messages processed, last-message timestamp)

```shell
ksql> DESCRIBE EXTENDED  FOO;
```

Print data from topic 

```shell
ksql> PRINT 'FOO' FROM BEGINNING;
```

Create Stream with non-default name topic   
the default topic name is stream name

```shell
ksql> CREATE STREAM LEEDS_USERS WITH (KAFKA_TOPIC='leeds-users') AS SELECT * FROM MOVEMENTS WHERE LCASE(LOCATION)='leeds' EMIT CHANGES;
```

# Aggregations

 ```shell
 ksql> CREATE TABLE PERSON_STATS WITH (VALUE_FORMAT='AVRO') AS SELECT ROWKEY AS PERSON, COUNT(*) AS LOCATION_CHANGES, COUNT_DISTINCT(LOCATION) AS UNIQUE_LOCATIONS FROM MOVEMENTS GROUP BY ROWKEY EMIT CHANGES;
 ```

Show Tables

```shell
ksql> show tables;
```

Query the Table

```shell
ksql> SELECT PERSON, LOCATION_CHANGES, UNIQUE_LOCATIONS FROM PERSON_STATS EMIT CHANGES;
```

Table as not always aggregates you can create table also directly from topic

```shell
ksql> CREATE TABLE MOVEMENTS_T (LOCATION VARCHAR) WITH (VALUE_FORMAT='JSON', KAFKA_TOPIC='movements');
```

# KSQLDB REST API
Fire curl command to get data

```shell
$  docker exec -it ksqldb curl -s -X "POST" "http://localhost:8088/query" \
-H "Content-Type: application/vnd.ksql.v1+json; charset=utf-8" \
-d '{"ksql": "SELECT PERSON, LOCATION_CHANGES, UNIQUE_LOCATIONS FROM PERSON_STATS WHERE ROWKEY='\''robin'\'';"}' | jq '.[].row' 
```


# Kafka Connector

check available connector

```shell
$ docker exec -it ksqldb curl -s localhost:80883/connector-plugins|jq '.[].class' 
```

SINK DATA DOWN TO A DATABASE

```shell
ksql> CREATE SINK CONNECTOR SINK_POSTGRES WITH (
    'connector.class'     = 'io.confluent.connect.jdbc.JdbcSinkConnector',
    'connection.url'      =  'jdbc:postgresql://postgres:5432/',
    'connection.user'     =  'postgres',
    'connection.password' =  'postgres',
    'topics'              =  'PERSON_STATS',
    'key.converter'       =  'org.apache.kafka.connect.storage.StringConverter',
    'auto.create'         =  'true',
    'insert.mode'         =  'upsert',
    'pk.mode'             =  'record_key',
    'pk.fields'           =  'PERSON'
);
```

show connectors
```shell
ksql> show connectors; 
```

# ENRICHING STREAMS
(basicall fancy name of saying some joining operations) 

```shell
ksql> CREATE STREAM ORDERS_ENRICHED AS SELECT O.*,I.*, O.ORDERUNITS * I.UNIT_COST AS TOTAL_ORDER_VALUE, FROM ORDERS O INNER JOIN ITEMS I ON O.ITEMID = I.ID;
```
the above query has data like this

```json
{
    "id: "Item_9",
    "make": "Boyle-McDermott",
    "model": "Apiacease",
    "unit_cost": 19.9
}
```

```json
{
    "ordertime": 1560070133853,
    "orderid": 67,
    "itemid": "Item_9",
    "orderunits": 5
}
```
so tha enriched output be

```json
{
    "ordertime": 1560070133853,
    "orderid": 67,
    "itemid": "Item_9",
    "orderunits": 5,
    "make": "Boyle-McDermott",
    "model": "Apiaceae",
    "unit_cost": 19.9,
    "total_order_value": 99.5
}
```

# CREATE A SOURCE CONNECTOR
```shell
ksql> CREATE SOURCE CONNECTOR SOURCE_MYSQL_01 WITH (
    'connector.class' = 'i.d.c.mysql.MySqlConnector',
    'database.hostname' = 'mysql',
    'table.whitelist'  = 'demo.customers'
);
```

eg: sink elastic search connector

```shell
ksql> CREATE SINK CONNECTOR SINK_ELESTIC_01 WITH (
   'connector.class' =  '...ElasticsearchConnector',
   'connection.url' = 'http://elasticsearch:9200',
   'topics' = 'orders'
);
```


#  Our use Case 
(ussdevents) stream

create an example stream
```shell
ksql>  CREATE STREAM ussd_event_stream (clientid VARCHAR, traceid VARCHAR) WITH (VALUE_FORMAT='JSON', PARTITIONS=1, KAFKA_TOPIC='ussdevents');
```

create table fo active users

```shell
ksql> CREATE TABLE USSD_STATS AS SELECT 1, COUNT_DISTINCT(clientid) FROM USSD_EVENT_STREAM WINDOW TUMBLING (SIZE 30 SECONDS) GROUP BY 1 EMIT CHANGES
```

query stored state
```shell
ksql> select KSQL_COL_1 from USSD_STATS WHERE  KSQL_COL_0=1 EMIT CHANGES;
```

```shell
ksql> CREATE TABLE USSD_CON_STATS WITH (VALUE_FORMAT='JSON') AS SELECT traceid, action FROM USSD_EVENT_STREAM GROUP BY traceid EMIT CHANGES;
```

```text
  started sessions
  session = action=end
  tracdeid = session
  <> (con * 100)/ total = bounded <>


  try this
   CREATE TABLE USSD_CON_STATS AS SELECT traceid, action, COUNT_DISTINCT(traceid) as TOTAL FROM USSD_EVENT_STREAM GROUP BY traceid, action EMIT CHANGES;

   table from topic
   CREATE TABLE USSD_CON_STATS (traceid VARCHAR PRIMARY KEY, action VARCHAR) WITH (VALUE_FORMAT='JSON', KAFKA_TOPIC='ussdevents');
```


what we have so far: changes /   *5/1/2021*


```text
CREATE TABLE USSD_CON_STATS (TRACEID STRING PRIMARY KEY, ACTION STRING) WITH (KAFKA_TOPIC='ussdevents', KEY_FORMAT='KAFKA', VALUE_FORMAT='JSON');
```

```text
CREATE TABLE USSD_CON_STATS (ID VARCHAR PRIMARY KEY, TRACEID VARCHAR, ACTION VARCHAR, timestamp varchar) WITH (KAFKA_TOPIC='ussdevents', KEY_FORMAT='KAFKA', VALUE_FORMAT='JSON');
```


```text
ksql> CREATE TABLE USSD_CON_STATS (ID VARCHAR PRIMARY KEY, TRACEID VARCHAR, ACTION VARCHAR, TIME VARCHAR) WITH (KAFKA_TOPIC='ussdevents', KEY_FORMAT='KAFKA', VALUE_FORMAT='JSON', timestamp='Time', timestamp_format='yyyy-MM-dd HH:mm:ss');
```



*works*
```text
ksql> CREATE TABLE USSD_CON_STATS (ID VARCHAR PRIMARY KEY, TRACEID VARCHAR,CLIENTID VARCHAR, ACTION VARCHAR) WITH (KAFKA_TOPIC='ussdevents', KEY_FORMAT='KAFKA', VALUE_FORMAT='JSON');
```



CREATE STREAM USSD_EVENT_STREAM (CLIENTID STRING, TRACEID STRING, ACTION STRING) WITH (KAFKA_TOPIC='ussdevents', KEY_FORMAT='KAFKA', PARTITIONS=1, VALUE_FORMAT='JSON')




*notwork*
```text
CREATE TABLE USSD_CON_STATS AS SELECT ID, TRACEID, CLIENTID, COUNT_DISTINCT(CLIENTID) FROM USSD_EVENT_STREAM GROUP BY CLIENTID EMIT CHANGES;
```


