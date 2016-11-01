# Cassandra Data Modeling Training

These are my notes from the Cassandra Data Modeling training from DataStax.

### Creating keyspaces

A keyspace is a container similar to databases (MSSQL) or schemas (Oracle).  This is where you can define replication factor.

``` sql
CREATE KEYSPACE killrvideo
WITH REPLICATION = { 
  'class': 'SimpleStrategy',
  'replication_factor': 1
};
```

### Switing to keyspaces 

``` sql
USE killrvideo;
```

### Basic Data Types

Text - UTF8 encoded string. Varchar is just an alias for text.
Int - Signed, 32 bit integer
UUID - universally unique identifier.  Third octet starts with a 4 (e.g. 52b11d6d-16e2-4ee2-b2a9-5ef1e9589328)
TIMEUUID - embeds a TIMESTAMP value.  This is sortable.  Third octet starts with a 1 (e.g. 1be43390-9fe4-11e3-8d05-425861b86ab6)
Timestamp - Stores date and time.  64-bit integer.  Milliseconds since epoch (1/1/1970).  Displays in cqlsh as yyyy-mm-dd HH:mm:ssZ

### Basic CREATE table query

``` sql
CREATE TABLE users (
  user_id UUID,
  first_name TEXT,
  last_name TEXT,
  PRIMARY KEY (user_id)
);
```

### Importing/Exporting CSV files using COPY

``` sql
COPY table1 (column1, column2, column3) FROM 'tabledata.csv' WITH HEADER=true;
```

### Basic SELECT queries

```sql
SELECT * 
FROM table1;

SELECT column1, column2, column3
FROM table1;

SELECT COUNT(*)
FROM table1;

SELECT *
FROM table1
LIMIT 10;
```

### Primary keys

* The primary key is compromised of the partition key and clusterd columns.  
* The primary key should contain all columns required to make the row unique.
* The partition key is how data is physically stored on disk.  All columns in the partition key are required in the where clause to pull back data. So plan accordingly when identifying what columns are hit in queries against the table.
* Clustered columns are additional columns that make the row unique as well as columns that you want to be able to query against.  Note that the order of the clustered columns is important.  You can't query against the 2nd clustered column without also including the 1st clustered column.

``` sql
CREATE TABLE users (
  user_id INT,
  first_name TEXT,
  last_name TEXT,
  phone_number TEXT,
  address TEXT,
  PRIMARY KEY ((last_name), first_name, phone_number, user_id )
);
```

* In the above example, the partition key is last_name.  
* The data will be grouped and stored on disk by last_name.  
* Any query against the table with a where clause must include last_name.  
* First_name, phone_number, and user_id columns are the clustered columns. Those can also be used in the where clause.  
* Note that phone_number can't be used without also using first_name because first_name comes before phone_number.



### Upserts

* When an insert is performed using the same primary key as an existing record, the new version of the record is also written (and becomes the active version of the data).  Instead of a traditional RDBMS that would throw a primary key violation, this instead writes a new data record which in turn becomes the most recent version of that row.
* Similarly, when you do an update for a record that doesn't exist, instead of updating zero records like a traditional RDBMS, a new record is created for that new primary key.

``` sql
CREATE TABLE employee (
 emp_id INT,
 emp_name TEXT,
 emp_address TEXT,
 emp_phone_number TEXT,
 PRIMARY KEY (emp_id)
);
```

* Given the above table, if you ran the following commands to load a single row of data and then write a similar row of data but only change the address, you will end up with a single row of data using the updated address (124 Main St).

``` sql
insert into employee (emp_id, emp_name, emp_address) values (1, 'John Doe', '123 Main St');
``` 
| emp_id | emp_name | emp_address | emp_phone_number |
|--------|----------|-------------|------------------|
| 1 | John Doe | 123 Main St | null |

``` sql
insert into employee (emp_id, emp_name, emp_address) values (1, 'John Doe', '124 Main St');
``` 

| emp_id | emp_name | emp_address | emp_phone_number |
|--------|----------|-------------|------------------|
| 1 | John Doe | 124 Main St | null |

* If you similarly ran the following command for a emp_id that doesn't exist, it would actually create a new record for that emp_id instead of updating zero records.

``` sql
update employee set emp_name = 'Jane Doe' where emp_id = 6;
```

| emp_id | emp_name | emp_address | emp_phone_number |
|--------|----------|-------------|------------------|
| 1 | John Doe | 124 Main St | null |
| 6 | Jane Doe | null | null |

* So why is this?  Because a traditional RDBMS has to read the exisitng data/index to verify if it exists or not, it is inheriently slower.  Cassandra doesn't do a read lookup beforehand, so it just writes the data out no matter if the same primary key row exists or not.  **Note this behavior can be altered if required.**  But upserts will occur by default.
