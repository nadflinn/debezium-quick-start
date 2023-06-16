
### Bring up Containers

```bash
DEBEZIUM_VERSION=1.9 docker-compose -f docker-compose-mysql.yaml up -d
```

Note: I have heard from people with M1 macs that the Kafka Connect image fails to build.

### Start Debezium
Shell into the kafka connect container:

```bash
docker exec -it kafka-connect bash
```

Start Debezium:

```bash
curl -i -X PUT -H "Accept:application/json" -H  "Content-Type:application/json" localhost:8083/connectors/inventory-connector/config -d '
{
  "connector.class": "io.debezium.connector.mysql.MySqlConnector",
  "tasks.max": "1",
  "database.hostname": "mysql",
  "database.port": "3306",
  "database.user": "debezium",
  "database.password": "dbz",
  "database.server.id": "184054",
  "database.server.name": "dbserver1",
  "database.include.list": "inventory",
  "database.history.kafka.bootstrap.servers": "kafka:19092",
  "database.history.kafka.topic": "dbhistory.inventory",
  "snapshot.mode": "schema_only",
  "provide.transaction.metadata": "true",
  "key.converter": "org.apache.kafka.connect.json.JsonConverter",
  "key.converter.schemas.enable": "false",
  "value.converter": "org.apache.kafka.connect.json.JsonConverter",
  "value.converter.schemas.enable": "false"
}'
```


### Create Record in MySQL
Shell into MySQL container:

```bash
docker exec -it mysql bash
```

Access MySQL cli
```bash
mysql -h 0.0.0.0 -uroot -pdebezium
```

Insert a record in customers table
```mysql
> use inventory
> INSERT INTO customers VALUES (1010, 'Joe', 'Smith', 'joe.smith@abc.com');
```

### Tail Kafka topic and view message

Shell into Kafka container:
```bash
docker exec -it kafka bash
```

Tail topic:
```bash
/kafka/bin/kafka-console-consumer.sh --bootstrap-server kafka:19092 --from-beginning --topic dbserver1.inventory.customers
```

You should see:
```json lines
{
  "before": null,
  "after": {
    "id": 1010,
    "first_name": "Joe",
    "last_name": "Smith",
    "email": "joe.smith@abc.com"
  },
  "source": {
    "version": "1.9.7.Final",
    "connector": "mysql",
    "name": "dbserver1",
    "ts_ms": 1686939506000,
    "snapshot": "false",
    "db": "inventory",
    "sequence": null,
    "table": "customers",
    "server_id": 1,
    "gtid": null,
    "file": "mysql-bin.000003",
    "pos": 488,
    "row": 0,
    "thread": 12,
    "query": null
  },
  "op": "c",
  "ts_ms": 1686939506733,
  "transaction": {
    "id": "file=mysql-bin.000003,pos=236",
    "total_order": 1,
    "data_collection_order": 1
  }
}
{
  "before": {
    "id": 1010,
    "first_name": "Joe",
    "last_name": "Smith",
    "email": "joe.smith@abc.com"
  },
  "after": {
    "id": 1010,
    "first_name": "Joe",
    "last_name": "Smith",
    "email": "joe.smith@def.com"
  },
  "source": {
    "version": "1.9.7.Final",
    "connector": "mysql",
    "name": "dbserver1",
    "ts_ms": 1686939682000,
    "snapshot": "false",
    "db": "inventory",
    "sequence": null,
    "table": "customers",
    "server_id": 1,
    "gtid": null,
    "file": "mysql-bin.000003",
    "pos": 1353,
    "row": 0,
    "thread": 12,
    "query": null
  },
  "op": "u",
  "ts_ms": 1686939682270,
  "transaction": {
    "id": "file=mysql-bin.000003,pos=1104",
    "total_order": 1,
    "data_collection_order": 1
  }
}
```


### Bring containers down
```bash
DEBEZIUM_VERSION=1.9 docker-compose -f docker-compose-mysql.yaml down
```