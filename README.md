# Tool to evaluate RDS MySQL downtime during RDS modification

The tool connects to the specified RDS MySQL instance and executes the same query (reading available collations) every second and prints results.

If the DB is unavailable, the tool keeps reconnecting to the DB every second during one minute.

The tool prints messages on stdout wit timestamps showing when the DB is available and when the DB is not available, so the user can tell for what period the DB was unavailable.

Usage:

```bash
dotnet tool -- -e <db-endpoint> -u <username> -p <password> -d <database> -l <label>
```

To assess the DB availability via different endpoints run the tool against RDS endpoint, Always-ON listener endpoint, and RDS Proxy endpoint in separate terminal windows:

* via **RDS endpoint**:

```bash
dotnet tool -- -e <rds-endpoint> -u <username> -p <password> -l RDS
```

* via **Always-ON listener endpoint**:

```bash
dotnet tool -- -e <alwayson-listener-endpoint> -u <username> -p <password> -l AlwaysON
```

NOTE: first time it took 5-7 connection retires till the tool was able to connect to the Always-ON listener in my case. Thereafter the tool connected to the endpoint instantaneously.

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
./rds-resize.sh database-1 db.m6i.xlarge
```

Keep watching the tool output to detect the downtime.

I resized my instance from `db.m6i.large` to `db.m6i.xlarge` (the operation took ~25 min) and got following results regarding the DB downtime:

* RDS endpoint: downtime 3 sec;

* Always-ON Listener endpoint: no downtime;

* RDS Proxy endpoint: no downtime.

NOTE: according to <https://learn.microsoft.com/en-us/sql/database-engine/availability-groups/windows/listeners-client-connectivity-application-failover?view=sql-server-ver16>

```text
During a failover, when the primary replica changes, existing connections to the listener are disconnected and new connections are routed to the new primary replica.
```

Therefore it's surprising we didn't get the downtime with Always On and the existing DB connection continued to work.  

## Example results

Started the resize (scale-up) at 12:12, the resize took ~25min. The down time hit between 12:36:04 - 12:36:07.

* RDS endpoint:

Downtime b/w 12:36:05-12:36:08 = 3sec

```text
12/2/2022 12:36:04 AM : RDS Starting a query...
master SQL_Latin1_General_CP1_CI_AS
tempdb SQL_Latin1_General_CP1_CI_AS
model SQL_Latin1_General_CP1_CI_AS
msdb SQL_Latin1_General_CP1_CI_AS
rdsadmin SQL_Latin1_General_CP1_CI_AS
12/2/2022 12:36:05 AM : RDS Starting a query...
12/2/2022 12:36:05 AM : RDS ERROR: SHUTDOWN is in progress., retry #1
12/2/2022 12:36:06 AM : RDS Opening a DB connection to xxxxx
12/2/2022 12:36:06 AM : RDS Starting a query...
12/2/2022 12:36:06 AM : RDS ERROR: SHUTDOWN is in progress.
Login failed for user 'admin'.
Cannot continue the execution because the session is in the kill state.
A severe error occurred on the current command.  The results, if any, should be discarded., retry #2
12/2/2022 12:36:07 AM : RDS Opening a DB connection to xxxxxx
12/2/2022 12:36:08 AM : RDS Starting a query...
master SQL_Latin1_General_CP1_CI_AS
tempdb SQL_Latin1_General_CP1_CI_AS
model SQL_Latin1_General_CP1_CI_AS
msdb SQL_Latin1_General_CP1_CI_AS
rdsadmin SQL_Latin1_General_CP1_CI_AS
```

* Always On Listener Endpoint:

WARNING: The first time the client was able to connect to the DB after 5-8 attempts.

No downtime between 12:36:04 - 12:36:08 and in the rest of logs.

```text
12/2/2022 12:36:04 AM : AG Starting a query...
master SQL_Latin1_General_CP1_CI_AS
tempdb SQL_Latin1_General_CP1_CI_AS
model SQL_Latin1_General_CP1_CI_AS
msdb SQL_Latin1_General_CP1_CI_AS
rdsadmin SQL_Latin1_General_CP1_CI_AS
12/2/2022 12:36:05 AM : AG Starting a query...
master SQL_Latin1_General_CP1_CI_AS
tempdb SQL_Latin1_General_CP1_CI_AS
model SQL_Latin1_General_CP1_CI_AS
msdb SQL_Latin1_General_CP1_CI_AS
rdsadmin SQL_Latin1_General_CP1_CI_AS
12/2/2022 12:36:06 AM : AG Starting a query...
master SQL_Latin1_General_CP1_CI_AS
tempdb SQL_Latin1_General_CP1_CI_AS
model SQL_Latin1_General_CP1_CI_AS
msdb SQL_Latin1_General_CP1_CI_AS
rdsadmin SQL_Latin1_General_CP1_CI_AS
12/2/2022 12:36:07 AM : AG Starting a query...
master SQL_Latin1_General_CP1_CI_AS
tempdb SQL_Latin1_General_CP1_CI_AS
model SQL_Latin1_General_CP1_CI_AS
msdb SQL_Latin1_General_CP1_CI_AS
rdsadmin SQL_Latin1_General_CP1_CI_AS
12/2/2022 12:36:08 AM : AG Starting a query...
master SQL_Latin1_General_CP1_CI_AS
tempdb SQL_Latin1_General_CP1_CI_AS
model SQL_Latin1_General_CP1_CI_AS
msdb SQL_Latin1_General_CP1_CI_AS
rdsadmin SQL_Latin1_General_CP1_CI_AS
```

* RDS Proxy:

No downtime between 12:36:04 - 12:36:08 and in the rest of logs.

```text
12/2/2022 12:36:04 AM : PROXY Starting a query...
master SQL_Latin1_General_CP1_CI_AS
tempdb SQL_Latin1_General_CP1_CI_AS
model SQL_Latin1_General_CP1_CI_AS
msdb SQL_Latin1_General_CP1_CI_AS
rdsadmin SQL_Latin1_General_CP1_CI_AS
12/2/2022 12:36:05 AM : PROXY Starting a query...
master SQL_Latin1_General_CP1_CI_AS
tempdb SQL_Latin1_General_CP1_CI_AS
model SQL_Latin1_General_CP1_CI_AS
msdb SQL_Latin1_General_CP1_CI_AS
rdsadmin SQL_Latin1_General_CP1_CI_AS
12/2/2022 12:36:06 AM : PROXY Starting a query...
master SQL_Latin1_General_CP1_CI_AS
tempdb SQL_Latin1_General_CP1_CI_AS
model SQL_Latin1_General_CP1_CI_AS
msdb SQL_Latin1_General_CP1_CI_AS
rdsadmin SQL_Latin1_General_CP1_CI_AS
12/2/2022 12:36:07 AM : PROXY Starting a query...
master SQL_Latin1_General_CP1_CI_AS
tempdb SQL_Latin1_General_CP1_CI_AS
model SQL_Latin1_General_CP1_CI_AS
msdb SQL_Latin1_General_CP1_CI_AS
rdsadmin SQL_Latin1_General_CP1_CI_AS
12/2/2022 12:36:08 AM : PROXY Starting a query...
master SQL_Latin1_General_CP1_CI_AS
tempdb SQL_Latin1_General_CP1_CI_AS
model SQL_Latin1_General_CP1_CI_AS
msdb SQL_Latin1_General_CP1_CI_AS
rdsadmin SQL_Latin1_General_CP1_CI_AS
```

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

## References

* Amazon RDS Proxy for MS SQL Server <https://aws.amazon.com/about-aws/whats-new/2022/09/amazon-rds-proxy-rds-sql-server/>

* When to use RDS Proxy <https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/rds-proxy-planning.html>

* Using Amazon RDS Proxy <https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/rds-proxy.html>

* MS SQL Always ON <https://learn.microsoft.com/en-us/sql/database-engine/availability-groups/windows/overview-of-always-on-availability-groups-sql-server?view=sql-server-ver16>

* Always ON listener <https://learn.microsoft.com/en-us/sql/database-engine/availability-groups/windows/listeners-client-connectivity-application-failover?view=sql-server-ver16>

Some pieces of the code are borrowed from the following sources:

* <https://learn.microsoft.com/en-us/azure/azure-sql/database/connect-query-dotnet-core?view=azuresql>

* <https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_SQLServerMultiAZ.html>

## TODO

Create similar tests for: PgSQL, and MySQL:

* <https://zetcode.com/csharp/postgresql/> PgSQL. Collation queries <https://dba.stackexchange.com/questions/29943/how-to-determine-the-collation-of-a-table-in-postgresql>

* <https://mysqlconnector.net/tutorials/connect-to-mysql/> MySQL. Collation queries <https://database.guide/how-to-show-database-collation-mysql/>