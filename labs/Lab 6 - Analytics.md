**[Back to Agenda](./../README.md)**

Hands On DSE Analytics
--------------------

Spark is general cluster compute engine. You can think of it in two pieces: **Streaming** and **Batch**.
**Streaming** is the processing of incoming data (in micro batches) before it gets written to Cassandra (or any database).
**Batch** includes both data crunching code and **SparkSQL**, a hive compliant SQL abstraction for **Batch** jobs.

It's a little tricky to have an entire class run streaming operations on a single cluster, so if you're interested in dissecting a full scale streaming app, check out [THIS git](https://github.com/retroryan/SparkAtScale).  


#### **Prereq for this lab**

** You can find the files user_purchase.tar.gz under DataStaxDay/data/ **
Please unzip with

```
tar xvfz user_purchase.tar.gz

```

Start `cqlsh` commandline and follow the instructions.
```
use <your_keyspace>;

CREATE TABLE IF NOT EXISTS users(
    user_id uuid,
    date_of_creation timestamp,
    firstname text,
    lastname text,
    age int,
    sex text,
    email text,
    phones map<text,text>,
    address text,
    city text,
    state text,
    country text,
    company text,
    job text,
    PRIMARY KEY(user_id)
);

CREATE TABLE IF NOT EXISTS user_purchases(
    user_id uuid,
    date timestamp,
    item text,
    price double,
    quantity int,
    total double,
    currency text,
    payment_method text,
    PRIMARY KEY((user_id),date)
);

// load data to tables
COPY users(user_id,date_of_creation,firstname,lastname,age,sex,email,phones,address,city,state,country,company,job)  FROM './user_purchase/users.csv'  WITH HEADER=false AND DELIMITER='|' AND DATETIMEFORMAT='%Y-%m-%dT%H:%M:%S.%fz' AND INGESTRATE=100000;


COPY user_purchases(user_id,date,item,price,quantity,total,currency,payment_method) FROM './user_purchase/user_purchase.csv' WITH HEADER=false AND DELIMITER='|' AND DATETIMEFORMAT='%Y-%m-%dT%H:%M:%S.%fz' AND INGESTRATE=60000;
```



#### **Spark has a REPL we can play in:**

```

dse spark spark.ui.port=<Pick a random 4 digit number> --total-executor-cores 2 --executor-memory 1G

```

>Notice the spark.ui.port flag - Because we are on a shared cluster, we need to specify a radom port so we don't clash with other users. We're also setting max cores = 1 or else one job will hog all the resources.

Lets check the table structure from spark:

```

:show retailer   

```

** We want to answer following business questions:**
1. Give me all users whose total amount for a single purchase is greater than 1000.0 euro
2. Give me the top buyer (user whose sum of total price of all transactions) for each country (join & group by)


Try some unfamiliar CQL commands on that retailer data - common statitics like min, max, avg, count, sum on the column **total**:

** scala DataFrame example **
```
spark.table("<your keyspace name>.user_purchases").
  groupBy("user_id").
  agg(count(($"date")).alias("count"),
      avg(($"total")).alias("avg"),
      min(($"total")).alias("min"),
      max(($"total")).alias("max"),
      sum(($"total")).alias("sum")).show
```

** spark SQL example **
```

spark.sql("SELECT user_id, count(total) count, min(total), max(total), avg(total), sum(total) FROM <your keyspace name>.user_purchases GROUP BY user_id ORDER BY sum(total) desc").show

```

>  ** 1. Give me all users whose total amount for a single purchase is greater than 1000.0 euro **

```
spark.table("<your keyspace name>.user_purchases").filter($"total">1000).show

```
** Joining the user table in order to get the details of the user **
```
spark.table("<your keyspace name>.user_purchases").alias("purchase").
    filter($"total">800).join(
        spark.table("<your keyspace name>.users").alias("users"), col("users.user_id") === col("purchase.user_id"), "inner").
    select($"users.user_id", $"firstname", $"email", $"total").show

```
** Same as before with spark SQL **
```
val df = spark.sql("SELECT u.user_id, firstname, email, total FROM <your keyspace name>.user_purchases p, retailer.users u WHERE p.user_id == u.user_id AND total > 800")
```


### ** Now write the data to DSE tables **

Create the table in your keyspace and check
```

df.createCassandraTable(
    "<your keyspace name>",
    "stats",
    partitionKeyColumns = Some(Seq("user_id")),
    clusteringKeyColumns = Some(Seq("email")))

df.printSchmea

:show <your keyspace name>
```

Save the data to DSE table

```
// write df to dse table
df.write.
cassandraFormat("stats", "<your keyspace name>").
save()
```


#### ** Try to find a solution for the second query
and write the data back to DSE **
>2. Give me the top buyer (user whose sum of total price of all transactions) for each country (join & group by)**

To exit the REPL type ```:quit```

You can see a great demo of running thse steps inside the Zeppelin Notebook [here](https://github.com/victorcouste/zeppelin-spark-cassandra-demo/)

**[Back to Agenda](./../README.md)**
