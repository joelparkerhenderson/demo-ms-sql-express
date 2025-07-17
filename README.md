# Demo MS SQL Express

Demonstrate Microsoft SQL Express server.

## Get Microsoft SQL Express via Docker

<https://learn.microsoft.com/en-us/sql/linux/quickstart-install-connect-docker>

Pull:

```sh
docker pull mcr.microsoft.com/azure-sql-edge
```

Run:

```sh
docker run \
    --env "ACCEPT_EULA=1" \
    --env "MSSQL_PID=Developer" \
    --env "MSSQL_USER=SA" \
    --env "MSSQL_SA_PASSWORD=TooManySecrets123" \
    --publish 1433:1433 \
    --detach \
    --name=sql \
    mcr.microsoft.com/azure-sql-edge
```

<https://learn.microsoft.com/en-us/sql/linux/sql-server-linux-configure-environment-variables>

Environment variable meanings:

* `ACCEPT_EULA=1` accepts the end user license agreement.

* `MSSQL_USER=SA` sets the user to be the system administrator.

* `MSSQL_SA_PASSWORD=toomanysecrets` sets the system administrator password.

Argument meanings:

* `--publish 1433:1433` Publish a container's port(s) to the host.

* `--detach` Run container in background and print container ID.

* `--name=sql` Assign a name to the container.

## Get MS SQLCMD

<https://learn.microsoft.com/en-us/sql/tools/sqlcmd/sqlcmd-utility>

macOS with brew:

```sh
brew install sqlcmd
```

Old:

* `pip install mssql-cli`

* `brew install --cask azure-data-studio`

## sqlcmd introduction

<https://learn.microsoft.com/en-us/sql/tools/sqlcmd/sqlcmd-use-utility>

Basic options:

* `-S Server` identifies the instance of SQL Server to which sqlcmd connects.

* `-U Username`

* `-P Password`

* `-E specifies a trusted connection`

## Connect

Connect via the "sa" login which is short for "system administrator". This login is automatically added as a member of the sysadmin fixed server role and thus has all permissions on that instance and can perform any activity.

Select the version information:

```sh
sqlcmd -U sa -P TooManySecrets123 -Q "select @@version;"
```

You should see output like:

```sqlout
Microsoft Azure SQL Edge Developer (RTM) - 15.0.2000.1574 (ARM64)
```

List the databases:

```sh
sqlcmd -U sa -P TooManySecrets123 -Q "select name from sys.databases;"
```

You should see output like:

```sqlout
name
------
master
tempdb
model
msdb
```

Create a database:

```sh
sqlcmd -U sa -P TooManySecrets123 -Q "create database demo;"
```

Verify your database exists:

```sh
sqlcmd -U sa -P TooManySecrets123 -Q "select name from sys.databases where name='demo';"
```

You should see output like:

```sqlout
name
----
demo
```

## Connect interactively

Run:

```sh
sqlcmd -S localhost -U sa -P TooManySecrets123
```

You should see the SQL prompt:

```sql
1>
```

You can type anything you want, and when you're ready to run it, then type "go" and return:

```sql
1> select @@version;
2> go
```

To quit, press ctrl-c.

## Create a table

Create:

```sql
create table my_table (my_date date, my_int int);
create index idx_my_date on my_table(my_date);
create index idx_my_int on my_table(my_int);
```

Verify:

```sql
select * from information_schema.tables;
```

## Insert a row

Insert:

```sql
insert into my_table values ('20000101', 10);
```

Verify:

```sql
select * from my_table;
```

You should see output like:

```sqlout
my_date          my_int
---------------- -----------
      2000-01-01          10
```

## Generate random data via functions

Create some procedures to generate random data:

```sql
-- Create a view to hold one random value.
-- This is necessary because the SQl RAND() function
-- has a side effect, so can't be used in a typical
-- user-defined function; instead it must be stored.
--
-- Example:
--
--     select r from random
--
create view random
as
select rand(checksum(newid())) as r
go

-- Generate a random date >= min and < max.
--
-- Example:
--
--     select dbo.random_date('20100101', '20200101');
--     go
--     2015-12-31
--
create function random_date (
    @min date,
    @max date
)
returns date as
begin
    return dateadd(day, (select r from random) * datediff(day,  @min, @max), @min)
end
go

-- Generate a random int >= min and < max.
--
-- Example:
--
--     select dbo.random_int(10, 20);
--     go
--     15
--
create function random_int (
    @min int,
    @max int
)
returns int as
begin
    return floor((select r from random) * (@max - @min) + @min)
end
go
```

## Generate random data via stored procedures

If you prefer to create your randomness capabilities via stored procedures, instead of via functions, then use this:

```sql
-- Generate a random date >= min and < max.
--
-- Example:
--
--     exec rand_date '20100101', '20200101';
--     go
--     2015-12-31
--
create procedure random_date
    @min date,
    @max date
as
    select dateadd(day, rand(checksum(newid())) * datediff(day, @min, @max), @min)
go

-- Generate a random int >= min and < max.
--
-- Example:
--
--     exec rand_int 10, 20;
--     go
--     15
--
create procedure random_int
    @min int,
    @max int
as
    select round(rand(checksum(newid())) * (@max - @min) + @min, 0)
go
```

## Insert

You can insert a table row with random data by using the random functions from above like:

```sql
insert into my_table values (
    (select dbo.random_date('20100101', '20200101')),
    (select dbo.random_int(10,20))
)
```

View:

```sql
select * from my_table;
go
```

Output is something like this:

```sqlout
my_date          my_int
---------------- -----------
      2015-12-31          15
```

## Insert loop

You can insert multiple table rows with random data by using a loop like:

```sql
declare @i int = 0
while (@i < 10)
begin
    insert into my_table values (
        (select dbo.random_date('20100101', '20200101')),
        (select dbo.random_int(10,20))
    )
    set @i = @i + 1
end
go
```

View:

```sql
select * from my_table;
go
```

Output is something like this:

```sqlout
my_date          my_int
---------------- -----------
      2011-05-26          18
      2012-04-14          15
      2016-12-19          12
      2017-10-14          12
      2011-03-25          12
      2012-03-29          17
      2018-11-23          11
      2013-09-01          11
      2015-11-29          13
      2010-10-02          12
```

## Benchmark

We can now use the random data to benchmark some SQL code, so we can experiment with ways to optimize it.

Delete any existing data:

```sql
delete from my_table;
go
```

Seed the table with a hundred thousand rows:

```sql
declare @i int = 0
while (@i < 100000)
begin
    insert into my_table values (
        (select dbo.random_date('20100101', '20200101')),
        (select dbo.random_int(10,20))
    )
    set @i = @i + 1
end
go
```

Turn on statistics time:

```sql
set statistics time on;
go
```

The statistics should show output like:

```sqlout
SQL Server parse and compile time:
   CPU time = 3 ms, elapsed time = 3 ms.

SQL Server Execution Times:
   CPU time = 99 ms,  elapsed time = 99 ms.
```

Run the example code that we want to benchmark:

```sql
select
    left(convert(varchar, my_date), 7) as my_month,
    count(*) as my_count
from
    my_table
group by
    left(convert(varchar, my_date), 7)
go
```

Output:

```sqlout
   CPU time = 48 ms,  elapsed time = 12 ms.
```

Run the experimental code that we want to benchmark:

```sql
select
    year(my_date),
    month(my_date),
    count(*) as my_count
from
    my_table
group by
    year(my_date),
    month(my_date)
go
```

Output:

```sqlout
 SQL Server Execution Times:
   CPU time = 45 ms,  elapsed time = 11 ms.
```

Turn off statistics time:

```sql
set statistics time off
go
```

**Conclusion: the implementations' execution times are nearly identical.**

## Docker wrap up

When you're completely finished with MS SQL Express, then use Docker to stop the container then remove it:

Run:

```sh
docker stop sql
docker remove sql
```
