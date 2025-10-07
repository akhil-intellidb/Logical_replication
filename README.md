**Logical replication in PostgreSQL**.

---

### 🧩 **1) Configure Publisher**

Edit the **`postgresql.conf`** file (on the Publisher node):

```bash
wal_level = logical
max_replication_slots = 10
max_wal_senders = 10
```

**Explanation:**

* `wal_level = logical` → enables logical replication.
* `max_replication_slots` → number of concurrent replication slots allowed (each subscriber uses one slot).
* `max_wal_senders` → max number of concurrent connections sending WAL data.

---

### 🔐 **2) Configure Access in `pg_hba.conf`**

Add the following lines to allow replication connections from the subscriber:

```
host  intellidb   replicator   <subscriber_ip>/32   md5
host  replication replicator   <subscriber_ip>/32   md5
```

Or use `trust` if you are in a test environment (no password required).
**Example:**

```
host  intellidb   replicator   192.168.56.103/32   md5
host  replication replicator   192.168.56.103/32   md5
```

---

### 🔁 **3) Reload or Restart PostgreSQL**

After changing configuration files, reload/restart the PostgreSQL service:

```bash
sudo systemctl restart intellidb
```

or if only `pg_hba.conf` was changed:

```bash
sudo systemctl reload intellidb
```

---

### 👤 **4) Create Replicator Role on Publisher**

If you used `md5` authentication:

```sql
CREATE ROLE replicator WITH LOGIN REPLICATION PASSWORD 'replica_pass';
```

If you used `trust` authentication:

```sql
CREATE ROLE replicator WITH LOGIN REPLICATION;
```

This role will be used by the subscriber to connect and replicate data.

---

### 📢 **5) Create Publication on Publisher**

A **publication** defines which tables or schemas will be replicated:

```sql
CREATE PUBLICATION my_publication FOR ALL TABLES;
```

You can also replicate specific tables:

```sql
CREATE PUBLICATION my_publication FOR TABLE employees, departments;
```

---

### 🧾 **6) Grant SELECT Privileges**

Replication works by reading data using SELECT, so you must grant access:

```sql
GRANT SELECT ON ALL TABLES IN SCHEMA public TO replicator;
```

Also ensure new tables get SELECT access automatically:

```sql
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO replicator;
```

---

### 🔗 **7) Create Subscription on Subscriber**

Run this **on the Subscriber node**:

```sql
CREATE SUBSCRIPTION my_subscription
CONNECTION 'host=<publisher_ip> port=5555 dbname=intellidb user=replicator password=replica_pass'
PUBLICATION my_publication
WITH (copy_data = true);
```

**Parameters explained:**

* `copy_data = true` → copies existing data when first created.
* `PUBLICATION my_publication` → must match the publication name on the publisher.

**Example:**

```sql
CREATE SUBSCRIPTION my_subscription
CONNECTION 'host=192.168.56.102 port=5555 dbname=intellidb user=replicator password=replica_pass'
PUBLICATION my_publication
WITH (copy_data = true);
```

---

### ✅ **8) Verify Replication**

Now test replication works:

**On Publisher:**

```sql
CREATE TABLE test (id serial primary key, name text);
INSERT INTO test (name) VALUES ('data_from_publisher');
```

**On Subscriber:**

```sql
SELECT * FROM test;
```

You should see:

```
 id |        name
----+--------------------
  1 | data_from_publisher
```

---

### 🔄 **Optional: Refresh Publication**

If you add new tables later, run:

```sql
ALTER SUBSCRIPTION my_subscription REFRESH PUBLICATION;
```

This updates the subscription with any new tables added to the publication.

---

