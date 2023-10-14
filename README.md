# outflux-demo
Demonstrate outflux schema migration from InfluxDB v1 to Timescale DB using Docker containers.

Follow the directions in the README here to install outflux: https://github.com/timescale/outflux

## Dependencies
- Docker
- outflux 
- pgcli

## Steps
1. Spin up the docker containers.
```shell
docker compose up -d
```
Verify that both Influx and Timescale containers are running.
```shell
docker ps
```

Successful output will look something like this:
```shell
CONTAINER ID   IMAGE                               COMMAND    				CREATED       		STATUS 			   PORTS 		NAMES 
b5ebc022a09c   timescale/timescaledb:latest-pg15   "docker-entrypoint.s…"   About an hour ago   Up About an hour   0.0.0.0:5432->5432/tcp  timescaledb
17c2ca5b0bf9   influxdb:1.7           			   "/entrypoint.sh infl…"   About an hour ago   Up About an hour   0.0.0.0:8083->8083/tcp, 0.0.0.0:8086->8086/tcp, 0.0.0.0:8088->8088/tcp   outflux-demo-influxdb-1
```

2. Write some sample data into Influx. 

Reference: [Influx docs](https://docs.influxdata.com/influxdb/v1/guides/write_data/#write-data-using-the-influxdb-api)

In the query below, we're writing to a measurement `reading` in the db `demo`. The tags are `reading_id` and `other_id`, the field value is `0.64` and the timestamp nanosecond epoch is `1696965565000000000`. Remember, when writing to Influx, the measurement will be created if it doesn't exist already.
```shell
curl -i -XPOST 'http://localhost:8086/write?db=demo&u=local&p=local' \
--data-binary 'reading,reading_id=123456789,other_id=abcdefg value=0.64 1696965565000000000'
```
3. Verify that the written data is queryable. 

Reference [Influx Docs](https://docs.influxdata.com/influxdb/v1/guides/query_data/#query-data-with-influxql)
```shell
curl -G 'http://localhost:8086/query?db=demo&u=local&p=local' \
--data-urlencode "q=SELECT \"value\" FROM \"reading\" WHERE \"reading_id\"='123456789'"
```

A successful query will result in output like this:
```json
{"results":[{"statement_id":0,"series":[{"name":"reading","columns":["time","value"],"values":[["2023-10-10T19:19:25Z",0.64]]}]}]}
```

4. Migrate the Influx schema into Timescale.

Reference [outflux readme](https://github.com/timescale/outflux). 

In the command below, the input server is Influx and the `output-conn` values refer to the Timescale db. For more info about the options used, see the reference link. 
```shell
# cd into whatever directory you've installed outflux
./outflux schema-transfer demo --input-server=http://localhost:8086 --input-user root --input-pass root --output-conn="user=postgres password=postgres host=0.0.0.0 port=5432 dbname=defaultdb sslmode=disable" --schema-strategy CreateIfMissing
```

A succesful command will result in output which ends with a line like this: `Schema Transfer complete in: 0.060 seconds`.

5. Verify the schema was migrated into Timescale.
```bash
# connect to the Timescale instance with pgcli
pgcli postgres://postgres:postgres@0.0.0.0:5432/defaultdb

# describe the new table created by the migration
 \d reading

```

Successful output will look like this:
```sql
postgres@0:defaultdb> \d reading
+-----------------+--------------------------+-----------+
| Column          | Type                     | Modifiers |
|-----------------+--------------------------+-----------|
| time            | timestamp with time zone |  not null |
| reading_id 	  | text                     |           |
| other_id        | text                     |           |
| value           | double precision         |           |
+-----------------+--------------------------+-----------+
Indexes:
    "reading_time_idx" btree ("time" DESC)
Triggers:
    ts_insert_blocker BEFORE INSERT ON reading FOR EACH ROW EXECUTE FUNCTION _timescaledb_internal.insert_blocker()

Time: 0.026s
```

6. Migrate the Influx data into Timescale.

Reference [outflux readme](https://github.com/timescale/outflux). 

In the command below, the input server is Influx and the `output-conn` values refer to the Timescale db. The options `from` and `to` are used to demonstrate a specified export timeframe. For more info about the options used, see the reference link. 

```shell
./outflux migrate demo reading --input-server=http://localhost:8086 --input-user root --input-pass root --output-conn="user=postgres password=postgres host=0.0.0.0 port=5432 dbname=defaultdb sslmode=disable" --from=2023-10-01T00:00:00Z --to=2023-11-01T00:00:00Z

```

A succesful command will result in output which ends with a line like this: `Migration execution time: 0.000 seconds`.

7. Verify the records were migrated into Timescale.

```bash
# connect to the Timescale instance with pgcli
pgcli postgres://postgres:postgres@0.0.0.0:5432/defaultdb

# select the new records
select * from reading;

```

Successful output will look like this:
```sql
+------------------------+----------+------------+-------+
| time                   | other_id | reading_id | value |
|------------------------+----------+------------+-------|
| 2023-10-10 19:19:25+00 | abcdefg  | 123456789  | 0.64  |
+------------------------+----------+------------+-------+
SELECT 1
Time: 0.018s
```

8. Shut down the docker containers
```shell
docker compose down
```