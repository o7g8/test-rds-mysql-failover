# Tool to evaluate RDS MySQL downtime during RDS modification

The tool connects to the specified RDS MySQL instance and executes the same query (reading available collations) every second and prints results.

If the DB is unavailable, the tool keeps reconnecting to the DB every second during one minute.

The tool prints messages on stdout wit timestamps showing when the DB is available and when the DB is not available, so the user can tell for what period the DB was unavailable.

Usage:

```bash
dotnet tool -- -e <db-endpoint> -u <username> -p <password> -d <database> -l <label>
```

To assess the DB availability via different endpoints run the tool against RDS endpoint, and RDS Proxy endpoint in separate terminal windows:

* via **RDS endpoint**:

```bash
dotnet tool -- -e <rds-endpoint> -u <username> -p <password> -l RDS
```

* via **RDS Proxy endpoint**:

```bash
dotnet tool -- -e <rdsproxy-endpoint> -u <username> -p <password> -l RDSProxy
```

Then modify the RDS database, e.g. resize the instance:

```bash
./rds-resize.sh <instance-identifier> <instance-size>
```

E.g. run:

```bash
./rds-resize.sh database-1 db.t3.large
```

Keep watching the tool output to detect the downtime.

I resized my instance from `db.t3.small` to `db.t3.large` (started 9:41-9:51 - the operation took ~10 min) and got following results regarding the DB downtime:

* RDS endpoint: downtime 9:49:33 - 9:50:38 = 65 sec

```text
12/6/2022 9:49:33 AM : RDS Starting a query...
12/6/2022 9:49:33 AM : RDS ERROR: Server shutdown in progress, retry #1
12/6/2022 9:49:34 AM : RDS Opening a DB connection to xxxxx
12/6/2022 9:49:34 AM : RDS ERROR: Unable to connect to any of the specified MySQL hosts., retry #2
...
12/6/2022 9:50:37 AM : RDS ERROR: Connect Timeout expired., retry #35
12/6/2022 9:50:38 AM : RDS Opening a DB connection to xxxxx
12/6/2022 9:50:38 AM : RDS Starting a query...
utf8mb4 utf8mb4_0900_ai_ci
utf8 utf8_general_ci
utf8mb4 utf8mb4_0900_ai_ci
utf8mb4 utf8mb4_0900_ai_ci
...
```

Note that the client made 35 reconnection attempts during the DB downtime.

* RDS Proxy endpoint: 9:50:06 - 9:50:39 = 33 sec:

```text
12/6/2022 9:49:33 AM : PROXY Starting a query...
12/6/2022 9:50:06 AM : PROXY ERROR: The Command Timeout expired before the operation completed., retry #1
12/6/2022 9:50:07 AM : PROXY Opening a DB connection to proxy-1670316268233-database-1.proxy-ckgfwrbmhiis.us-east-1.rds.amazonaws.com
12/6/2022 9:50:39 AM : PROXY Starting a query...
utf8mb4 utf8mb4_0900_ai_ci
utf8 utf8_general_ci
utf8mb4 utf8mb4_0900_ai_ci
utf8mb4 utf8mb4_0900_ai_ci
```

Note that it was only single reconnect attempt, which took 32 sec.  

## Aurora/MySQL

### Resize writer instance

```bash
./rds-resize.sh database-2-instance-1 db.t3.medium
```

NOTE: the resized writer will become a new reader and the previous reader will be promoted to be a writer.

* Writer endpoint - downtime 1:30:19-1:30:27 (8 sec)

```text
12/6/2022 1:30:19 PM : WRITER Starting a query...
12/6/2022 1:30:19 PM : WRITER ERROR: Failed to read the result set., retry #1
12/6/2022 1:30:20 PM : WRITER Opening a DB connection to database-2.cluster-ckgfwrbmhiis.us-east-1.rds.amazonaws.com
...
12/6/2022 1:30:26 PM : WRITER ERROR: Unable to connect to any of the specified MySQL hosts., retry #8
12/6/2022 1:30:27 PM : WRITER Opening a DB connection to xxxxx
12/6/2022 1:30:27 PM : WRITER Starting a query...
utf8 utf8_general_ci
latin1 latin1_swedish_ci
utf8 utf8_general_ci
utf8 utf8_general_ci
...
```

* Reader endpoint - downtime 1:30:19-1:30:27 (8 sec)

```text
12/6/2022 1:30:19 PM : READER Starting a query...
12/6/2022 1:30:19 PM : READER ERROR: Failed to read the result set., retry #1
12/6/2022 1:30:20 PM : READER Opening a DB connection to xxxxx
...
12/6/2022 1:30:26 PM : READER ERROR: Unable to connect to any of the specified MySQL hosts., retry #8
12/6/2022 1:30:27 PM : READER Opening a DB connection to xxxxx
12/6/2022 1:30:27 PM : READER Starting a query...
utf8 utf8_general_ci
latin1 latin1_swedish_ci
utf8 utf8_general_ci
utf8 utf8_general_ci
```

* Proxy writer endpoint: no downtime!

```text
...
12/6/2022 1:30:19 PM : PROXY_WRITER Starting a query...
utf8 utf8_general_ci
latin1 latin1_swedish_ci
utf8 utf8_general_ci
utf8 utf8_general_ci
12/6/2022 1:30:20 PM : PROXY_WRITER Starting a query...
utf8 utf8_general_ci
latin1 latin1_swedish_ci
utf8 utf8_general_ci
utf8 utf8_general_ci
...
```

* Proxy reader endpoint: downtime 2:09:46-2:13:09 (3min 23 sec)

```text
...
12/6/2022 2:09:46 PM : PROXY_READER Starting a query...
12/6/2022 2:10:18 PM : PROXY_READER ERROR: The Command Timeout expired before the operation completed., retry #1
12/6/2022 2:10:19 PM : PROXY_READER Opening a DB connection to xxxxx
12/6/2022 2:12:19 PM : PROXY_READER ERROR: Connect Timeout expired., retry #2
12/6/2022 2:12:20 PM : PROXY_READER Opening a DB connection to xxxxx
12/6/2022 2:13:09 PM : PROXY_READER Starting a query...
utf8 utf8_general_ci
latin1 latin1_swedish_ci
utf8 utf8_general_ci
utf8 utf8_general_ci
...
```

NOTE: very strange to get the downtime from the reader endpoint!!! Need to try to create a cluster with a dedicated reader endpoint.

### Resize reader instance

We resize the new reader:

```bash
./rds-resize.sh database-2-instance-1 db.t3.small
```

* Writer endpoint: no downtime

* Reader endpoint: downtime 1:50:27-1:50:28 (1 sec)

```text
12/6/2022 1:50:27 PM : READER Starting a query...
12/6/2022 1:50:27 PM : READER ERROR: Failed to read the result set., retry #1
12/6/2022 1:50:28 PM : READER Opening a DB connection to database-2.cluster-ro-ckgfwrbmhiis.us-east-1.rds.amazonaws.com
12/6/2022 1:50:28 PM : READER Starting a query...
utf8 utf8_general_ci
latin1 latin1_swedish_ci
utf8 utf8_general_ci
utf8 utf8_general_ci
```

* Proxy writer endpoint: no downtime

* Proxy reader endpoint: downtime 1:50:27-1:54:50 (4min 23 sec!!!)

```text
...
12/6/2022 1:50:27 PM : PROXY_READER Starting a query...
12/6/2022 1:50:59 PM : PROXY_READER ERROR: The Command Timeout expired before the operation completed., retry #1
12/6/2022 1:51:00 PM : PROXY_READER Opening a DB connection to proxy-1670332417056-database-2-read-only.endpoint.proxy-ckgfwrbmhiis.us-east-1.rds.amazonaws.com
12/6/2022 1:53:00 PM : PROXY_READER ERROR: Connect Timeout expired., retry #2
12/6/2022 1:53:01 PM : PROXY_READER Opening a DB connection to proxy-1670332417056-database-2-read-only.endpoint.proxy-ckgfwrbmhiis.us-east-1.rds.amazonaws.com
12/6/2022 1:54:50 PM : PROXY_READER Starting a query...
utf8 utf8_general_ci
latin1 latin1_swedish_ci
utf8 utf8_general_ci
utf8 utf8_general_ci
```

NOTE: very strange to get the extremely long downtime from the reader endpoint!!! Need to try to create a cluster with a dedicated reader endpoint.

## Conclusions

With the default Aurora production setup you also get 2 nodes for HA (similarly to RDS Multi-AZ), but unlike with Multi-AZ you can use the 2nd node for read-only operations.
If  you can refactor your software to split DB reads from writes (make them access different DB endpoints), then you can scale DB “write” and “read” instances independently. And unlike with RDS Multi-AZ the Writer and Reader instances in Aurora can have different sizes.

Overall the failover times with Aurora are a lot better compared to RDS MySQL (though I need to investigate why RDS Proxy ReadOnly endpoint has the anomaly long timeouts).

## Build

Install .NET Core 6 (or better) following a guide for your OS, e.g. <https://learn.microsoft.com/en-us/dotnet/core/install/linux-ubuntu>

Build the tool:

```bash
dotnet build
```

## Create RDS MySQL database

* RDS > Create Database

* Standard Create

* Engine type: MySQL

* Edition: MySQL Community

* Engine Version: 8.0.28 (pick the latest one)

* Templates: Production

* Deployment options: Multi-AZ DB instance (try Cluster as well)

* DB instance identifier: `<choose your own>`

* Master username: admin

* Master Password: `<choose your own>`

* DB instance class: Standard classes, `db.t3.small` (or `<choose your own>`)

* Storage type: `gp3`

* Allocated storage: `20Gb`

* Enable storage autoscaling: Yes

* Multi-AZ deployment: Yes (Mirroring / Always On)

* Don't connect to an EC2 compute resource.

* VPC: `<choose your own>`

* Public access: No

* VPC security group: Create new. NB! You will need to edit the security group associated with the RDS instance to allow traffic from your EC2 instance running the "downtime detection" tool.

* Create an RDS Proxy: Yes

Create an EC2 and allow it to connect to the RDS in the RDS security group.

## Create RDS Aurora/MySQL database

* RDS > Create Database

* Standard Create

* Engine type: Amazon Aurora

* Edition: Amazon Aurora MySQL-Compatible Edition

* Engine Version: Aurora (MySQL 5.7) (pick the latest one)

* Templates: Production

* DB cluster identifier: `<choose your own>`

* Master username: `admin` (or `<choose your own>`)

* Master Password: `<choose your own>`

* DB instance class: Standard classes, `db.t3.small` (or `<choose your own>`)

* Multi-AZ deployment: Don't create an Aurora Replica

* Don't connect to an EC2 compute resource.

* VPC: `<choose your own>`

* Public access: No

* VPC security group: Create new. NB! You will need to edit the security group associated with the RDS instance to allow traffic from your EC2 instance running the "downtime detection" tool.

* Create an RDS Proxy: Yes

* Database authentication: Password authentication

Create an EC2 and allow it to connect to the RDS in the RDS security group.

## References

* When to use RDS Proxy <https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/rds-proxy-planning.html>

* Using Amazon RDS Proxy <https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/rds-proxy.html>

Some pieces of the code are borrowed from the following sources:

* <https://learn.microsoft.com/en-us/azure/azure-sql/database/connect-query-dotnet-core?view=azuresql>

* <https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_SQLServerMultiAZ.html>

## TODO

Create similar tests for: PgSQL:

* <https://zetcode.com/csharp/postgresql/> PgSQL. Collation queries <https://dba.stackexchange.com/questions/29943/how-to-determine-the-collation-of-a-table-in-postgresql>
