# dremio-iceberg-demo
Demonstrate the Dremio Data Lake engine accessing Apache Iceberg tables stored in HDFS

# Introduction
The purpose of this demo is to show how Dremio can access Apache Iceberg table formatted data sets in HDFS. This is a very low level demo and would not be appropriate for a first call or early discovery meeting.

## Pre-requisites

- Dremio Data Lake Engine version 14.x
- Apache Hadoop 3.x running at least HDFS and the Hive metastore
- Spark 3.0.x with Apache Iceberg JAR files
- An SSH client

## OR

- Launch a Dremio playground environment using these instructions: <Dremio Playground - General Purpose Lab Environment>

# The Demo

### Step 1. Configure Dremio to read Apache Iceberg tables

DO: As a Dremio administrator, run the following commands in a Dremio “New Query” SQL editor:

```
ALTER SYSTEM SET dremio.iceberg.enabled = true;
ALTER SYSTEM SET reflection.manager.use_iceberg_dataset.enabled = true;
SELECT name, bool_val, num_val FROM sys.options WHERE name like '%iceberg%'
```

SAY: In this version of Dremio, the ability to read Apache Iceberg tables is a preview feature and is disabled by default. Let's run some commands to enable Dremio to read Iceberg tables.

### Step 2. Create a  warehouse location in HDFS for the Iceberg tables

**DO**: Open an SSH session to the Dremio Coordinator node and use the Hadoop DFS commands to create a new location in HDFS. As a user that has permissions to create HDFS directories (such as the “hdfs” user in the Dremio playground environment, run the following commands: 

```
su - hdfs

hadoop fs -mkdir /iceberg_warehouse
hadoop fs -chown -R spark:hadoop /iceberg_warehouse

```

SAY: Let's create a place to store our new Apache Iceberg tables, using the HDFS file system that is available. 

### Step 3.  Use a Spark-SQL session to create the Apache Iceberg tables

**DO**: In the SSH session to the Dremio Coordinator node, su to a user that has permissions to run Spark jobs and access HDFS. In the Dremio playground, the “spark” user can be used. Then start the spark-sql shell.

```
su - spark

spark-sql --packages org.apache.iceberg:iceberg-spark3-runtime:0.11.0 org.apache.spark.sql.sources.v2 \
          --conf spark.sql.catalog.iceberg_catalog=org.apache.iceberg.spark.SparkCatalog \
          --conf spark.sql.catalog.iceberg_catalog.type=hadoop \
          --conf spark.sql.catalog.iceberg_catalog.warehouse=hdfs:///iceberg_warehouse
```

SAY: We have a Spark environment running in clustered mode and we are going to create the Apache Iceberg tables using the Spark-SQL shell. Here is the command for starting the Spark SQL shell. Notice that we told the Spark SQL shell to include the Iceberg package and to reference the newly created iceberg_warehouse location in HDFS.

**DO**: In the spark-sql shell, run the following DDL command:

```
--
-- Create the Customers Iceberg table
--
CREATE DATABASE iceberg_catalog.iceberg_db;
USE iceberg_catalog.iceberg_db;

CREATE TABLE customers_iceberg_table (
    customer_id bigint NOT NULL,
    created_date string NOT NULL,
    company_name string,
    contact_person string,
    contact_phone string,
    active boolean
)    
USING iceberg;
```

SAY: Let's pretend that we are hosting software for our customers and we will be tracking server log events in our Iceberg tables. First, we will create a customer table to store some information about our customers, including customer name and main contact. Notice we specified “USING iceberg” in our DDL to cause Spark to use the Apache Iceberg table formats.

**DO**: In the spark-sql shell, run the following DML commands:


```
--
-- Add some sample data to the customer table
--
INSERT INTO customers_iceberg_table VALUES (4001, "2019-12-15", "Acme Manufacturing", "Jay Blakeley", "415-555-1212", true);
INSERT INTO customers_iceberg_table VALUES (4002, "2018-12-14", "Top Pharma Corp.", "Jessica Smith", "415-555-3652", true);
INSERT INTO customers_iceberg_table VALUES (4003, "2014-12-17", "Safe Investors, Inc.", "Vijay Pandha", "415-555-6845", true);
INSERT INTO customers_iceberg_table VALUES (4004, "2013-12-15", "Northeast Life Insurance, Inc.", "Foster Bing", "415-555-9487", true);

SELECT * FROM customers_iceberg_table;
```

SAY: Let’s put some sample data in this new customer table. We will give each a unique id and  customer name.

**DO**: In the spark-sql shell, run the following DDL commands:

```
--
-- Create the log events Iceberg table
--
CREATE TABLE log_events_iceberg_table (
    event_id bigint NOT NULL,
    customer_id bigint NOT NULL,
    event_date string NOT NULL,
    event_timestamp string NOT NULL,
    hostname string,
    log_entry string,
    audited boolean
)
USING iceberg
PARTITIONED BY (event_date);
```

SAY: Now let’s create the main log_events table to store system events from the servers that we are running. We will include a place to put customer_id so we can join with the other table later.  Notice that we specified the “USING iceberg” clause again? And we also specified a “PARTITIONED BY” clause to simulate a real environment, where we would use partitioned data sets to improve performance.

**DO**: In the spark-sql shell, run the following DML commands:

```
--
-- Add some sample data to the log_events table
--
INSERT INTO log_events_iceberg_table VALUES (1, 4001, "2020-12-15", "2020-12-15 03:12:00.000", "129.5.3.200", "User greg signed in.", false);
INSERT INTO log_events_iceberg_table VALUES (2, 4001, "2020-12-15", "2020-12-15 03:13:00.000", "129.5.3.201", "User pat signed in.", false);
INSERT INTO log_events_iceberg_table VALUES (3, 4002, "2020-12-15", "2020-12-15 03:14:00.000", "129.5.3.201", "User charlie signed in.", false);
INSERT INTO log_events_iceberg_table VALUES (4, 4003, "2020-12-16", "2020-12-16 03:10:00.000", "129.5.3.200", "User mike signed in.", false);
INSERT INTO log_events_iceberg_table VALUES (5, 4003, "2020-12-16", "2020-12-16 03:11:00.000", "129.5.3.201", "User stanley signed in.", false);
INSERT INTO log_events_iceberg_table VALUES (6, 4003, "2020-12-16", "2020-12-16 03:12:00.000", "129.5.3.201", "User ravi signed in.", false);
INSERT INTO log_events_iceberg_table VALUES (7, 4004, "2020-12-16", "2020-12-16 03:13:00.000", "129.5.3.200", "User sarah signed in.", false);
INSERT INTO log_events_iceberg_table VALUES (8, 4004, "2020-12-17", "2020-12-17 03:22:00.000", "129.5.3.231", "User frank signed in.", false);
INSERT INTO log_events_iceberg_table VALUES (9, 4004, "2020-12-17", "2020-12-17 03:23:00.000", "129.5.3.231", "User marge signed in.", false);
```
SAY: Let’s put some sample data into the log_events table. 

**DO**: Open up a second SSH session to your Dremio Coordinator node and run the following commands: 

```
hadoop fs -ls -R /iceberg_warehouse | less -S
```

SAY: Just to show you where the Iceberg tables are being stored, let’s run an HDFS command to show what files where created in the iceberg warehouse location. Notice that it is not just data files, but it also has metadata files to keep track of versions and schema information.

### Step 4.  Connect Dremio to the Apache Iceberg tables

**DO**: In the Dremio Web UI, create a new HDFS source that points to your HDFS instance. Specify the following in the data source and save the new source.

```
Name: hdfs

Namenode: <the private IP address of your Dremio Coordinator node>

Metadata: Fetch every 1 minute

Metadata: Expire after 1 minute
```

SAY: Now let’s setup a new data lake data source in Dremio to access the Apache Iceberg tables on HDFS.

**DO**: Click on the new “hdfs” source and navigate to the hdfs->iceberg_warehouse->iceberg_db location. Then select the “Format Folder” option for the customers_iceberg_table object. Select the dropdown box showing the other formats that Dremio supports. Finally save the VDS in the Preparation semantic layer tier.

SAY: As I navigate to one of the new Iceberg tables, Dremio can create a physical dataset reference to it. Notice how Dremio automatically sensed that it was an Apache Iceberg table?  If it was something else, then Dremio would have most likely sense that as well. Let’s save it as a Dremio virtual dataset in one of our semantic layers, the preparation tier. I will name it “customers”.

**DO**: Do the same for the log_events Iceberg table. Navigate to the hdfs->iceberg_warehouse->iceberg_db location. Then select the “Format Folder” option for the log_events_iceberg_table object. Save the VDS in the Preparation semantic layer tier and name it “log_events”.

SAY: Now I will do the same for the log_events table, and save it in the preparation semantic layer.

**DO**: While still in the newly saved log_events VDS, click on the Join button and joint it with the preparation.customers VDS via the customer_id primary/foreign key relationship. Save the newly joined dataset in the Business semantic layer and name it customer_log_events. After the customer_log_events VDS is saved, keep it displayed and show how Dremio query results.

SAY: It might be useful to our business users if we join these two tables together so they can easily use it in a BI report or data science workload. I will use Dremio’s join function to create a third virtual dataset that relates these two Iceberg tables via the customer_id fields. I will save it in a semantic layer that is above the preparation tier, I will save it in the Business tier.

### Step 6.  Use Tableau to demonstrate a BI tool querying the Iceberg tables via Dremio

**DO**: From within the customer_log_events VDS click on the Tableau toolbar button to generate a .tds file and then click on the .tds file and launch a Tableau workbook. Log in to Dremio via the Tableau log in dialog.

SAY: Now let's see how a business user would use Dremio to access those Apache Iceberg tables in HDFS. I am launching a Tableau session that will connect live to our Dremio data lake engine. Notice how it prompts me for my AD/LDAP credentials?

**DO**: Within the Tableau workbook, drag the following columns from the left side of the workbook to the “rows” section of the sheet.

```
Company Name

Event Timestamps

Log Entry
```

SAY: With our live connection from Tableau to Dremio,  Tableau has no idea that Dremio is querying the Iceberg tables. To Tableau it looks just like any other ANS SQL data source. You can see the customers and log_entries data being added to this worksheet.

### Step 7.  Update the Iceberg tables

**DO**: In the spark-sql shell, run the following DML commands.  When you created the “hdfs” data source, you specified a metadata refresh period of 1 minute. It should table about that long for Dremio to “see” this new data.

```
--
-- Add more data to the log_events table
--
INSERT INTO log_events_iceberg_table VALUES (10, 4002, "2020-12-18", "2020-12-18 02:13:00.000", "129.5.3.201", "User pinky signed in.", false);
INSERT INTO log_events_iceberg_table VALUES (11, 4001, "2020-12-18", "2020-12-18 04:22:00.000", "129.5.3.225", "User edward signed in.", false);
INSERT INTO log_events_iceberg_table VALUES (12, 4003, "2020-12-19", "2020-12-19 07:23:00.000", "129.5.3.225", "User sriva signed in.", false);

SELECT * FROM log_events_iceberg_table ORDER BY event_timestamp;
```

SAY: Now let's add some data to our log_events table and see how Dremio can query the additional data.

**DO**: In the spark-sql shell, run the following DDL command to create a “changes” table for log events.  

```
--
-- Create an Iceberg table to track change operations (DELETE, UPDATE, INSERT)
--
CREATE TABLE log_events_iceberg_table_changes (
    operation string,
    event_id bigint NOT NULL,
    event_date string NOT NULL,
    event_timestamp string NOT NULL,
    hostname string,
    log_entry string,
    audited boolean
)    
USING iceberg;
```

SAY: With Apache Iceberg, it is possible to update tables using the “MERGE INTO” command. This allows an Iceberg table to be updated with new records, updated records and deleted record based on a “changes” reference table. Notice how there is a “operation” column added to the table? This will be used by the “MERGE INTO” logic to determine if the change is an update, insert or delete operation. This “changes” table would be produced by a data pipeline implemented by an ETL tool, or something like Spark, Flink or Kafka based pipelines.

**DO**:  In the spark-sql shell, run the following DML commands to simulate “changes” table for the log events.  

```
--
-- Add some sample data to the log_events "changes" table
--
INSERT INTO log_events_iceberg_table_changes 
  VALUES ('DELETE', 2, "2020-12-15", "2020-12-15 03:13:00.000", "129.5.3.201", "User pat signed in.", false);

INSERT INTO log_events_iceberg_table_changes 
  VALUES ('UPDATE', 5, "2020-12-16", "2020-12-16 03:11:00.000", "129.5.3.201", "User stanley signed in.", true);

INSERT INTO log_events_iceberg_table_changes 
  VALUES ('INSERT', 10, "2020-12-18", "2020-12-18 03:26:00.000", "129.5.3.211", "User sangita signed in.", false);

SELECT * FROM log_events_iceberg_table_changes;
```

SAY: We will manually enter some “changes” to simulate a data pipeline. Here we have one DELETE operation, one UPDATE operation and one INSERT operation.

**DO**: In the spark-sql shell, run the “MERGE INTO” command to update records in the main log_events Iceberg table.


```
--
-- Merge the change operations into the main Iceberg table
--
MERGE INTO log_events_iceberg_table t
  USING ( SELECT * FROM log_events_iceberg_table_changes s )
  ON t.event_id = s.event_id
    WHEN MATCHED AND s.operation = 'DELETE' THEN DELETE
    WHEN MATCHED AND s.operation = 'UPDATE' THEN UPDATE SET t.audited = s.audited
    WHEN NOT MATCHED THEN INSERT (event_id, event_date, event_timestamp, hostname, log_entry, audited) 
                             VALUES (event_id, event_date, event_timestamp, hostname, log_entry, audited);
```

SAY: This “MERGE INTO” command will implement the merge logic and determine if a change is an update, insert or delete operation. Unfortunately, Spark 3.0.x, the version we are running doesn’t support the “MERGE INTO” command, so we would have to implement this as a Java class or something similar.

**DO**: Nothing

SAY: So what we just saw was how easy it is for Dremio to access Apache Iceberg tables, and how easy it is to create virtual datasets including joined tables using Dremio. And how BI users and data scientists can easily access Iceberg tables when connected to the Dremio data lake engine.

**DO**: In the spark-sql shell, run the following DDL commands to remove the Iceberg tables.

```
--
--  Drop the iceberg tables 
--
DROP TABLE iceberg_db.customers_iceberg_table;
DROP TABLE iceberg_db.log_events_iceberg_table;
DROP TABLE iceberg_db.log_events_iceberg_table_changes;

DROP DATABASE iceberg_db;
```

SAY: Nothing

**DO**: In the SSH session, run the following command to remove the Iceberg warehouse location in HDFS.

```
# Delete the HDFS warehouse
hadoop fs -rm -r /iceberg_warehouse
```

SAY: Nothing

---

Direct comments or questions to greg@dremio.com

 
