# 🐞 Apache Flink + Kafka Integration – Error Log & Solutions

> 💡 This document collects all **real-world errors** I faced when integrating Flink with Kafka (inside Docker), and provides **concrete solutions** to each.

---

## ❌ ERROR 1: `Timed out waiting for a node assignment. Call: describeTopics`

**Cause:**  
Flink couldn't reach the Kafka broker to fetch topic metadata.

**Why it happens:**  
Using `localhost:9092` from inside a Flink container doesn’t work because `localhost` refers to the container itself.

**✅ Solution:**  
Use Docker network hostname:
```sql
'properties.bootstrap.servers' = 'kafka:29092'
```

---

## ❌ ERROR 2: `Failed to get metadata for topics [test-topic]`

**Cause:**  
Flink is unable to find the Kafka topic due to connection or naming issues.

**✅ Solution:**
- Ensure topic exists:
  ```bash
  docker exec -it kafka kafka-topics --bootstrap-server localhost:9092 --list
  ```
- Use correct topic name in the SQL:
  ```sql
  'topic' = 'test-topic'
  ```
- Use correct broker host:
  ```sql
  'properties.bootstrap.servers' = 'kafka:29092'
  ```

---

## ❌ ERROR 3: Flink Job fails with:
`org.apache.flink.runtime.JobException: Recovery is suppressed by NoRestartBackoffTimeStrategy`

**Cause:**  
Flink job failed and cannot be restarted automatically.

**✅ Solution:**
- Open Flink Web UI → View logs of the failed job.
- Fix root cause (typically Kafka connection or parsing).
- Rerun the job manually.

---

## ❌ ERROR 4: Flink SQL Client shows no data from Kafka topic

**Cause:**  
Data was sent to Kafka **before** starting the `INSERT INTO` in Flink.

**✅ Solution:**
- Always run:
  ```sql
  INSERT INTO sink_table SELECT * FROM kafka_source_table;
  ```
- Then send data to Kafka topic
- Make sure table uses:
  ```sql
  'scan.startup.mode' = 'earliest-offset'
  ```

---

## ❌ ERROR 5: Flink Web UI shows job failed with:
`Failed to list subscribed topic partitions`

**Cause:**  
Flink can't get topic partitions info from Kafka (metadata issue).

**✅ Solution:**
- Use correct Kafka hostname: `kafka:29092`
- Make sure topic exists
- Restart Flink if needed

---

## ❌ ERROR 6: Kafka console consumer shows repeated messages or errors

**Cause:**  
Invalid format or encoding issue when producing messages from Windows CMD.

**✅ Solution:**
Use this for Windows CMD:
```bash
echo "{"id":101,"name":"Flink","email":"flink@test.com","created_at":1650000000000}" | docker exec -i kafka kafka-console-producer --broker-list localhost:9092 --topic test-topic
```

---

## ❌ ERROR 7: `INSERT INTO` Flink → No data reaches PostgreSQL Sink

**Possible Causes:**
- Duplicate primary keys (`id`)
- You didn’t send new data after job started
- Table mapping doesn’t match schema
- PostgreSQL connector not loaded

**✅ Solution:**
- Make sure `id` is **unique**.
- Ensure sink table is correct:
  ```sql
  CREATE TABLE postgres_sink (
    id INT,
    name STRING,
    email STRING,
    created_at BIGINT
  ) WITH (
    'connector' = 'jdbc',
    'url' = 'jdbc:postgresql://postgres:5432/outbox_demo',
    'table-name' = 'customers_sink',
    'username' = 'postgres',
    'password' = 'postgres'
  );
  ```
- Add connector JARs to `/opt/flink/lib/`:
  - `flink-connector-jdbc-*.jar`
  - `flink-connector-jdbc-postgres-*.jar`
  - `postgresql-*.jar`

---

## ❌ ERROR 8: Table not found error in SQL client

**Cause:**  
You didn’t define the table in the current session.

**✅ Solution:**  
Always define the table again in a new session, e.g.:
```sql
CREATE TABLE test_kafka_json (...) WITH (...);
```

---

## ❌ ERROR 9: `json parse error` or empty table from Kafka source

**Cause:**  
Kafka messages contain invalid or partial JSON.

**✅ Solution:**  
Use:
```sql
'format' = 'json',
'json.ignore-parse-errors' = 'true'
```

---

## ❌ ERROR 10: Job stuck, no data flow visible

**Cause:**  
Job is running, but not receiving new records.

**✅ Solution:**
- Send new messages **after** job starts
- Use:
```sql
SELECT * FROM print_sink;
```
to debug

---

## ✅ Final Tips

- Use `kafka:29092` for Kafka inside Docker.
- Always start `INSERT INTO ...` before sending test data.
- If PostgreSQL doesn't receive data, make sure all `JAR` dependencies exist in `/opt/flink/lib/`.
- Watch Flink Web UI logs for job failures.
- Restart Flink session if tables disappear.

---

## 📌 Summary

These errors took **3 full days** to debug due to:
- Lack of examples with Flink 2.x
- Poor error messages
- Docker DNS issues
- Flink Job restart strategy set to `NoRestart`

This README is my attempt to save **future engineers** hours (or days) of struggle 💙
