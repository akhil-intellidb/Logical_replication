# pgEdge BDR Setup Documentation

**Date:** February 2024  
**Version:** pgEdge 24.10-10  

## Architecture Overview
- **Node1:** 192.168.56.101
- **Node2:** 192.168.56.102
- **Database:** bdrdb
- **Replication User:** bdruser
- **Replication Extension:** Spock

---

## Node1 Setup Steps

### 1. Initial Installation
```sh
cd ~
wget https://pgedge-download.s3.amazonaws.com/REPO/install.py
python3 install.py
```

### 2. PostgreSQL Setup
```sh
cd pgedge
./pgedge install pg16
./pgedge init pg16
```

### 3. Configure Network Access
```sh
echo "listen_addresses = '*'" >> ~/pgedge/data/pg16/postgresql.conf
echo "host all all 192.168.56.102/32 scram-sha-256" >> ~/pgedge/data/pg16/pg_hba.conf
```

### 4. Allow Firewall Access (If applicable)
```sh
sudo firewall-cmd --add-service=postgresql --permanent
sudo firewall-cmd --reload
```

### 5. Restart PostgreSQL to Apply Changes
```sh
./pgedge restart pg16
```

### 6. Database Setup
```sh
./pg16/bin/psql -U postgres -c "CREATE DATABASE bdrdb;"
./pg16/bin/psql -U postgres -c "CREATE ROLE bdruser WITH LOGIN SUPERUSER PASSWORD 'BDRpass123!';"
```

### 7. Create Node for Replication
```sh
./pg16/bin/psql -U postgres -d bdrdb -c "CREATE EXTENSION spock;"
./pg16/bin/psql -U postgres -d bdrdb -c "SELECT spock.node_create(node_name := 'node1', dsn := 'host=192.168.56.101 port=5432 dbname=bdrdb user=bdruser password=BDRpass123!');"
```

---

## Node2 Setup Steps

### 1. Initial Installation (Same as Node1)

### 2. PostgreSQL Setup (Same as Node1)

### 3. Configure Network Access
```sh
echo "listen_addresses = '*'" >> ~/pgedge/data/pg16/postgresql.conf
echo "host all all 192.168.56.101/32 scram-sha-256" >> ~/pgedge/data/pg16/pg_hba.conf
```

### 4. Allow Firewall Access (If applicable)
```sh
sudo firewall-cmd --add-service=postgresql --permanent
sudo firewall-cmd --reload
```

### 5. Restart PostgreSQL to Apply Changes
```sh
./pgedge restart pg16
```

### 6. Database Setup
```sh
./pg16/bin/psql -U postgres -c "CREATE DATABASE bdrdb;"
./pg16/bin/psql -U postgres -c "CREATE ROLE bdruser WITH LOGIN SUPERUSER PASSWORD 'BDRpass123!';"
```

### 7. Create Node for Replication
```sh
./pg16/bin/psql -U postgres -d bdrdb -c "CREATE EXTENSION spock;"
./pg16/bin/psql -U postgres -d bdrdb -c "SELECT spock.node_create(node_name := 'node2', dsn := 'host=192.168.56.102 port=5432 dbname=bdrdb user=bdruser password=BDRpass123!');"
```

### 8. Create Subscription to Node1
```sh
./pg16/bin/psql -U postgres -d bdrdb -c "SELECT spock.sub_create(subscription_name := 'sub_to_node1', provider_dsn := 'host=192.168.56.101 port=5432 dbname=bdrdb user=bdruser password=BDRpass123!');"
```

### 9. Create Subscription to Node2 (On Node1)
```sh
./pg16/bin/psql -U postgres -d bdrdb -c "SELECT spock.sub_create(subscription_name := 'sub_to_node2', provider_dsn := 'host=192.168.56.102 port=5432 dbname=bdrdb user=bdruser password=BDRpass123!');"
```

---

## Verification Commands

### Check Node Status
```sh
./pg16/bin/psql -U postgres -d bdrdb -c "SELECT * FROM spock.node;"
```

### Check Subscription Status
```sh
./pg16/bin/psql -U postgres -d bdrdb -c "SELECT * FROM spock.subscription;"
```

### Test Replication
```sh
./pg16/bin/psql -U postgres -d bdrdb -c "CREATE TABLE test_table (id SERIAL PRIMARY KEY, data TEXT);"
./pg16/bin/psql -U postgres -d bdrdb -c "INSERT INTO test_table (id, data) VALUES (1, 'Test data');"
```

---

## Maintenance Commands

### Restart PostgreSQL
```sh
./pgedge restart pg16
```

### Resync Tables
```sh
./pg16/bin/psql -U postgres -d bdrdb -c "SELECT spock.sub_resync_table('sub_to_node1', 'test_table');"
```

### Check Replication Status
```sh
./pg16/bin/psql -U postgres -d bdrdb -c "SELECT * FROM spock.sub_show_status();"
```

---

## Best Practices
- Use **odd IDs on Node1** and **even IDs on Node2** to prevent conflicts.
- Regularly monitor **replication lag** using `spock.sub_show_status()`.
- Perform **periodic data consistency checks** between nodes.
- Keep **PostgreSQL and Spock versions** in sync across nodes.

---

## Troubleshooting

### Check PostgreSQL Status
```sh
./pgedge status
```

### Verify Network Connectivity
```sh
pg_isready -h 192.168.56.101 -p 5432
pg_isready -h 192.168.56.102 -p 5432
```

### Check Log Files
```sh
ls -l ~/pgedge/data/logs/
cat ~/pgedge/data/logs/postgresql.log
```

### Verify User Permissions
```sh
./pg16/bin/psql -U postgres -d bdrdb -c "SELECT usename, usesuper FROM pg_user WHERE usename = 'bdruser';"
```

