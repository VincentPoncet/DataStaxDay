Welcome to DataStax Essentials Day!
===================
![icon](http://i.imgur.com/FoIOBlt.png)

In this session, you'll learn all about DataStax Enterprise. It's a mix between presentation and hands-on. This is your reference for the hands-on content. Feel free to bookmark this page for future reference! 

>Based on original work by DataStax colleagues Marc Selwan, Rich Reffner, Victor Coustenoble, Simon Ambridge and Negib Marhoul.

----------


Hands On Setup
-------------

We have a 3 node cluster for you to play with! The cluster is currently running in both **search** and **analytics** mode so you can take advantage of both Spark and Solr on your Cassandra data. 

You'll be given the connection details when the cluster starts.

Use the following IPs:
Each column in the room connect to one cluster

Column 1 => cluster 1

Column 2 => cluster 2
…

```
cluster 1	54.154.172.118
cluster 2	54.154.51.206
cluster 3	54.154.116.180
cluster 4	176.34.132.26
cluster 5	54.154.116.137
cluster 6	54.229.229.14
cluster 7	176.34.140.46
cluster 8	54.246.162.95
```

To SSH into the cluster, connect as root using the password provided and the external address of one of the nodes:
```
ssh root@[ipaddress] 
```

**NB** an external address is the one you use to connect to a node from outside the cluster. An internal address is the one that the machines use to communicate with each other.

Note down your IP addresses. You will be given the external addresses of the nodes in your cluster. You can obtain the internal addresses via the hosts file on any of the nodes e.g.
```
cat /etc/hosts
```
In this example we'll use these IP addresses:


You should then be able to connect to the management consoles for OpsCenter, Spark and Solr:

* OpsCenter: http://Node 0 external address:8888 
* Spark Master: http://Node 1 external address:7080 
* Solr UI: http://Node 0 external address:8983/solr 

OpsCenter and Solr should always start on Node 0, but you may need to check the node where the Spark Master is running. You can easily do this by connecting to one of the nodes via ssh and using the command:
```
dsetool sparkmaster
spark://172.31.24.137:7077
```

In this example the response tells us that the Spark Master is running on internal address 172.31.24.137 - so we need to use the corresponding external address http://54.213.132.207:7080 to acccess it from a browser on a client machine.


#### Connecting to the cluster from DevCenter
If you've installed DevCenter you can use it to run queries, view schemas, manage scripts etc. 

To access your cluster:
- Simply add a new connection, supplying a name for your connection and for the contact host use any of the external IP's provided.
- Wait a few seconds for the connection to complete, and the keyspace and table details for the database will be displayed
- You can view Keyspaces and tables, run CQL commands and save scripts.

You can run most of the following exercioses in DevCenter but to get the full benefit of the advanced features like consistency levels you should use **cqlsh** as an interactive command line tool for CQL access to Cassandra. 
For the exercies below we will be using cqlsh.

----------


Hands On DSE Cassandra 
-------------------

Cassandra is the brains of DSE. It's an awesome storage engine that handles replication, availability, structuring, and of course, storing the data at lightning speeds. It's important to get yourself acquainted with Cassandra to fully utilize the power of the DSE Stack. 

#### Creating a Keyspace, Table, and Queries 

Start cqlsh like this from the command prompt on one of the nodes in the cluster:

```
cqlsh <node private ip address>
``` 
or
```
cqlsh <node name> 
```
ie:
```
cqlsh node0
```

Let's make our first Cassandra Keyspace! 

```
CREATE KEYSPACE <Enter your firstname/name> WITH replication = {'class': 'SimpleStrategy', 'replication_factor': 3 };
```

And just like that, any data within any table you create under your keyspace will automatically be replicated 3 times.

> **Hint** - SimpleStrategy is OK for a cluster using a single data center, but in the real world with multiple datacenters you would use the ```NetworkTopologyStrategy``` replication strategy. In fact, even if you start out on your development path with just a single data center, if there is even a chance that you might go to multiple data centers in the future, then you should use NetworkTopologyStrategy from the outset.

Let's keep going and create ourselves a table.


```
CREATE TABLE <yourkeyspace>.sales (
	name text,
	time int,
	item text,
	price double,
	PRIMARY KEY (name, time)
) WITH CLUSTERING ORDER BY ( time DESC );
```

> Yup. This table is very simple but don't worry, we'll play with some more interesting tables in just a minute.

Let's get some data into your table! Cut and paste these inserts into DevCenter or CQLSH. Feel free to insert your own data values, as well. 

```
INSERT INTO <yourkeyspace>.sales (name, time, item, price) VALUES ('kunal', 20150205, 'Apple Watch', 299.00);
INSERT INTO <yourkeyspace>.sales (name, time, item, price) VALUES ('kunal', 20150204, 'Apple iPad', 999.00);
INSERT INTO <yourkeyspace>.sales (name, time, item, price) VALUES ('rich', 20150206, 'Music Man Stingray Bass', 1499.00);
INSERT INTO <yourkeyspace>.sales (name, time, item, price) VALUES ('kunal', 20150207, 'Jimi Hendrix Stratocaster', 899.00);
INSERT INTO <yourkeyspace>.sales (name, time, item, price) VALUES ('rich', 20150208, 'Santa Cruz Tallboy 29er', 4599.00);
```

At the moment we're prefixing the keyspace name to the table name in our CQL commands e.g. ```<yourkeyspace>.sales```.

Let's make it a little easier - we can set our ***default*** keyspace so that we dont need to type it in every time.

```
use <yourkeyspace>;
```
You can check the tables that are in that keyspace like this:
```
describe tables
```
> Of course, if there are tables with the **same name** in **other** keyspaces it may be wiser to continue to use a keyspace prefix to avoid inadvertently modifying the data in the wrong table!

We can check how many rows there are in our table after the insert of five rows:
```
select count(*) from <yourkeyspace>.sales;
```
or
```
select count(*) from sales;
```

> Be careful with ```count(*)``` - it will scan the entire cluster. This wouldnt be a good idea in a big cluster with millions or billions of rows!

To retrieve data:

```
SELECT * FROM sales where name='kunal' AND time >=20150207 ;
```
>See what I did there? You can do range scans on clustering keys! Give it a try.
>We will look at partition keys and clustering keys in the next section

----------


Hands On Cassandra Primary Keys 
-------------------

#### The secret sauce of the Cassandra data model: Primary Key

There are just a few key concepts you need to know when beginning to data model in Cassandra. But if you want to know the real secret sauce to solving your use cases and getting great performance, then you need to understand how Primary Keys work in Cassandra. 

Let's dive in! 

Since Cassandra use cases are typically focused on performance and up-time, it's critical to understand how Primary Key (PK) definition, query capabilities, and performance are related.

Here's how to do the exercise...

Remember to check that you're using your default keyspace:

```
use <yourkeyspace>;
```

1) Use the CQL script below to create the tables. You'll notice that all the tables are exactly the same except for the primary key definition.
```

create table sentiment1 (
body text,         // social message body text
dt int,            // date of message
ch text,           // social channel
cu text,           // customer
sent text,         // sentiment indicator
primary key (dt)
);

create table sentiment2 (
body text,
dt int,
ch text,
cu text,
sent text,
primary key ((ch,dt))
);

create table sentiment3 (
body text,
dt int,
ch text,
cu text,
sent text,
primary key (ch,dt)
);

create table sentiment4 (
body text,
dt int,
ch text,
cu text,
sent text,
primary key (ch,cu,dt)
);

create table sentiment5 (
body text,
dt int,
ch text,
cu text,
sent text,
primary key ((ch,dt),cu)
);

```

2) Use the CQL script below to populate the tables with data.
```
begin batch
insert into sentiment1 (body,dt,ch,cu,sent) values ('I''m feeling sick.',20160102,'twitter','red bull','negative');
insert into sentiment2 (body,dt,ch,cu,sent) values ('I''m feeling sick.',20160102,'twitter','red bull','negative');
insert into sentiment3 (body,dt,ch,cu,sent) values ('I''m feeling sick.',20160102,'twitter','red bull','negative');
insert into sentiment4 (body,dt,ch,cu,sent) values ('I''m feeling sick.',20160102,'twitter','red bull','negative');
insert into sentiment5 (body,dt,ch,cu,sent) values ('I''m feeling sick.',20160102,'twitter','red bull','negative');
apply batch;

begin batch
insert into sentiment1 (body,dt,ch,cu,sent) values ('That was sick!',20160101,'facebook','red bull','positive');
insert into sentiment2 (body,dt,ch,cu,sent) values ('That was sick!',20160101,'facebook','red bull','positive');
insert into sentiment3 (body,dt,ch,cu,sent) values ('That was sick!',20160101,'facebook','red bull','positive');
insert into sentiment4 (body,dt,ch,cu,sent) values ('That was sick!',20160101,'facebook','red bull','positive');
insert into sentiment5 (body,dt,ch,cu,sent) values ('That was sick!',20160101,'facebook','red bull','positive');
apply batch;

begin batch
insert into sentiment1 (body,dt,ch,cu,sent) values ('I feel sick, too.',20160102,'facebook','dew tour','negative');
insert into sentiment2 (body,dt,ch,cu,sent) values ('I feel sick, too.',20160102,'facebook','dew tour','negative');
insert into sentiment3 (body,dt,ch,cu,sent) values ('I feel sick, too.',20160102,'facebook','dew tour','negative');
insert into sentiment4 (body,dt,ch,cu,sent) values ('I feel sick, too.',20160102,'facebook','dew tour','negative');
insert into sentiment5 (body,dt,ch,cu,sent) values ('I feel sick, too.',20160102,'facebook','dew tour','negative');
apply batch;

begin batch
insert into sentiment1 (body,dt,ch,cu,sent) values ('Dude, you''re sick.',20160103,'facebook','red bull','positive');
insert into sentiment2 (body,dt,ch,cu,sent) values ('Dude, you''re sick.',20160103,'facebook','red bull','positive');
insert into sentiment3 (body,dt,ch,cu,sent) values ('Dude, you''re sick.',20160103,'facebook','red bull','positive');
insert into sentiment4 (body,dt,ch,cu,sent) values ('Dude, you''re sick.',20160103,'facebook','red bull','positive');
insert into sentiment5 (body,dt,ch,cu,sent) values ('Dude, you''re sick.',20160103,'facebook','red bull','positive');
apply batch;

begin batch
insert into sentiment1 (body,dt,ch,cu,sent) values ('How sick was that?',20160103,'facebook','monster energy','positive');
insert into sentiment2 (body,dt,ch,cu,sent) values ('How sick was that?',20160103,'facebook','monster energy','positive');
insert into sentiment3 (body,dt,ch,cu,sent) values ('How sick was that?',20160103,'facebook','monster energy','positive');
insert into sentiment4 (body,dt,ch,cu,sent) values ('How sick was that?',20160103,'facebook','monster energy','positive');
insert into sentiment5 (body,dt,ch,cu,sent) values ('How sick was that?',20160103,'facebook','monster energy','positive');
apply batch;
```

3) Look at these queries. For one table at a time, copy/paste/run the groups of queries. In other words, run all of the queries for sentiment1 at the same time. Check out Cassandra's response. Then run all queries for sentiment2 at the same time, etc. You'll notice that some of the queries work against some of the tables, but not all. Why?
```
select * from sentiment1 where dt = 20160102;
select * from sentiment1 where ch = 'facebook' and dt = 20160102;
select * from sentiment1 where ch = 'facebook' and dt > 20160101;

select * from sentiment2 where dt = 20160102;
select * from sentiment2 where ch = 'facebook' and dt = 20160102;
select * from sentiment2 where ch = 'facebook' and dt > 20160101;

select * from sentiment3 where dt = 20160102;
select * from sentiment3 where ch = 'facebook' and dt = 20160102;
select * from sentiment3 where ch = 'facebook' and dt > 20160101;
select * from sentiment3 where ch = 'facebook' and cu = 'red bull' and dt >= 20160102 and dt <= 20160103;
select * from sentiment3 where ch = 'facebook' and dt >= 20160102 and dt <= 20160103;

select * from sentiment4 where dt = 20160102;
select * from sentiment4 where ch = 'facebook' and cu = 'red bull' and dt >= 20160102 and dt <= 20160103;
select * from sentiment4 where ch = 'facebook' and dt >= 20160102 and dt <= 20160103;

select * from sentiment5 where dt = 20160102;
select * from sentiment5 where ch = 'facebook' and dt = 20160102;
select * from sentiment5 where ch = 'facebook' and dt > 20160101;
select * from sentiment5 where ch = 'facebook' and dt >= 20160102 and dt <= 20160103;
```

4) Extra Credit 1: Did the query "select * from sentimentX where ch = 'facebook' and dt >= 20160102 and dt <= 20160103;" work for any of the tables? Why or why not?

5) Extra Credit 2: What would you do if you needed to find all messages with positive sentiment?

6) Challenge Question: In the real world, how many tweets would you guess occur per day? As of this writing, Twitter generates ~500M tweets/day according to [these guys](http://www.internetlivestats.com/twitter-statistics/). Let's say we need to run a query that captures all tweets over a specified range of time. Given our data model scenario, we simply data model a primary key value of (ch, dt) to capture all tweets in a single Cassandra row sorted in order of time, right? Easy! But, alas, the Cassandra logical limit of single row size (2B columns in C* v2.1) would fill up after about 4 days! Our primary key won't work. What would we do to solve our query?

Have fun!

Cassandra Data Model and Query Pro-Tips
---------------------------------------

Here are a few Cassandra data modeling pro-tips and principles to stay out of trouble and get you moving the right direction:

**Primary Keys**: Know what a partition key is. Know what a clustering key is. Know how they work for storing the data and for allowing query functionality. This exercise is a great start.

**Secondary Indexes**: If you're tempted to use a secondary index in Cassandra in production, at least in Cassandra 2.1, don't do it. Instead, create a new table with a PK definition that will meet your query needs. In Cassandra, denormalization is fast and scalable. Secondary indexes aren't as much. Why? Lots of reason that have to do with the fact that Cassandra is a distributed system. It's a good thing.

**Relational Data Models**: Relational data models don't work well (or at all) in Cassandra. That's a good thing, because Cassandra avoids the extra overhead involved in processing relational operations. It's part of what makes Cassandra fast and scalable. It also means you should not copy your relational tables to Cassandra if you're migrating a relational system to Cassandra. Use a well-designed Cassandra data model.

**Joins**: Cassandra doesn't support joins. How do you create M:1 and M:M relations? Easy... denormalize your data model and use a PK definition that works. Think in materialized views. Denormalization is often a no-no in relational systems. To get 100% up-time, massive scale/throughput and speed that Cassandra delivers, it's the right way to go.

**Allow Filtering**: If you're tempted to use Allow Filtering in production, see the advice for Secondary Indexes above.

**Batches**: Batches solve a different problem in Cassandra than they do in relational databases. Use them to get an atomic operation for a single PK across multiple tables. Do NOT use them to batch large numbers of operations assuming Cassandra will optimize the query performance of the batch. It doesn't work that way. Use batches appropriately or not at all.


----------


Hands On Cassandra Consistency 
-------------------

#### Let's play with consistency!

Consistency in Cassandra refers to the number of acknowledgements that replica nodes need to send to the coordinator for an operation to be considered successful whilst also providing good data (avoiding dirty reads). 

We recommend a ** default replication factor of 3 and consistency level of LOCAL_QUORUM as a starting point**. You will almost always get the performance you need with these default settings.

In some cases, developers find Cassandra's replication fast enough to warrant lower consistency for even better latency SLA's. For cases where very strong global consistency is required, possibly across data centers in real time, a developer can trade latency for a higher consistency level. 

Let's give it a shot. 

**In CQLSH**:

```
tracing on
consistency all
```

Any query will now be traced. Cassandra provides a description of each step it takes to satisfy the request, the names of nodes that are affected, the time for each step, and the total time for the request.

**Consistency** level "all" means all 3 replicas need to respond to a given request (read OR write) to be successful. Let's do a **SELECT** statement to see the effects.

Set your keyspace created earlier as the default keyspace:
```
use <yourkeyspace>;
```
Retrieve all rows from the sales table where name="kunal":
```
SELECT * FROM sales where name='kunal';
```

How did we do? We returned three records. Total elapsed time for the request on our 3-node cluster was 16370 microseconds. 

```
Request complete | 2016-06-02 08:13:54.028370 |  172.31.30.32 |          16370
```

**Let's compare a lower consistency level:**

```consistency local_quorum```

>Quorum means majority: RF/2 + 1. In our case, 3/2 = 1 + 1 = 2. At least 2 nodes need to acknowledge the request. 

Let's try the **SELECT** statement again. Any changes in latency? Run it a couple of times for a consistent result. 

```
Request complete | 2016-06-03 05:26:04.894594 | 172.31.24.137 |           4594
```

This looks much better now doesn't it? **LOCAL_QUORUM** is the most commonly used consistency level among developers. It provides a good level of performance and a moderate amount of consistency. That being said, many use cases can warrant  **CL=LOCAL_ONE**. 

>Keep in mind that our dataset is so small, it's sitting in memory on all nodes. With larger datasets that spill to disk, the latency cost become much more drastic.

Let's try this again but this time we only want the fastest replica to respond. Pay attention to what's happening in the trace.

```
consistency local_one
```
```
SELECT * FROM sales where name='kunal';
```

Take a look at the trace output. Look at all queries and contact points. What you're witnessing is both the beauty and challenge of distributed systems.

And there's a big reduction in request latency too:

```
Request complete | 2016-06-03 05:27:57.145701 | 172.31.24.137 |           1701
```

For more detailed classes on data modeling, consistency, and Cassandra 101, check out the free classes at the [DataStax Academy](www.academy.datastax.com) website. 

----------


Hands On DSE Search
-------------
DSE Search is awesome. You can configure which columns of which Cassandra tables you'd like indexed in **lucene** format to make extended searches more efficient whilst enabling features such as text search and geospatial search. 

Let's start off by indexing the tables we've already made. Here's where the dsetool really comes in handy:

```
dsetool create_core <yourkeyspace>.sales generateResources=true reindex=true
```

>If you've ever created your own Solr cluster, you know you need to create the core and upload a schema and config.xml. That **generateResources** tag does that for you. For production use, you'll want to take the resources and edit them to your needs but it does save you a few steps. 

This by default will map Cassandra types to Solr types for you. Anyone familiar with Solr knows that there's a REST API for querying data. In DSE Search, we embed that into CQL so you can take advantage of all the goodness CQL brings. Let's give it a shot. 

Run the following commands in cqlsh:
```
select * from sales WHERE solr_query='{"q":"name: kunal"}';
```

Output should be like this:
```
 name  | time     | item                      | price | solr_query
-------+----------+---------------------------+-------+------------
 kunal | 20150207 | Jimi Hendrix Stratocaster |   899 |       null
 kunal | 20150205 |               Apple Watch |   299 |       null
 kunal | 20150204 |                Apple iPad |   999 |       null
```

Let's try a filter query to return only the items from Apple: 
```
select * from sales WHERE solr_query='{"q":"name:kunal", "fq":"item:*pple*"}'; 

 name  | time     | item        | price | solr_query
-------+----------+-------------+-------+------------
 kunal | 20150205 | Apple Watch |   299 |       null
 kunal | 20150204 |  Apple iPad |   999 |       null
```

We can control how the data is sorted based on a column value:
```
select * from sales WHERE solr_query='{"q":"name:kunal", "fq":"item:*pple*", "sort":"price desc"}';

 name  | time     | item        | price | solr_query
-------+----------+-------------+-------+------------
 kunal | 20150204 |  Apple iPad |   999 |       null
 kunal | 20150205 | Apple Watch |   299 |       null
 ```
 
> For your reference, [here's the doc](http://docs.datastax.com/en/datastax_enterprise/4.8/datastax_enterprise/srch/srchCql.html?scroll=srchCQL__srchSolrTokenExp) that shows some of things you can do.

OK! Time to work with some more interesting data. Meet **Amazon Book Sales** data.

There are some Linux and Cassandra pre-requisites need for this exercise.
* development tools like gcc compiler, Python libraries
* Pip Python - package manager
* DataStax Python Driver
 
NOTE : All python dependancies are already installed on DataStax Days clusters.

You can check if they're already installed using a package manager e.g. on Ubuntu:
```
apt-cache policy gcc python-dev python-pip python-dev build-essential
```
If any of the packages are not installed, follow the instructions below. If you're sharing a cluster with other students you should decide between you who will perform these tasks on which nodes!

**Install pip and dependencies**

```
sudo apt-get install gcc python-dev
sudo apt-get install python-pip python-dev build-essential 
sudo pip install --upgrade pip
sudo pip install --upgrade virtualenv 
```

**Install the Python Cassandra Driver**

The next step is to install the DataStax Cassandra Python Driver.

You can check if its already installed using the following command:
```
pip show cassandra-driver
---
Name: cassandra-driver
Version: 3.4.1
Location: /usr/local/lib/python2.7/dist-packages
Requires: six, futures
```
If it *isn't* already installed, use the following command to install it:

>This might take some time on less powerful machines

```
sudo pip install cassandra-driver
``` 

Now we need to load the data and create our Solr cores.

**Run solr_dataloader.py**

This will create the CQL schemas and load the data. Be sure to pass the name of your keyspace as a parameter:

```
./create_data.sh <name of your keyspace>
...
Loading into Keyspace ...
loading geo
loading meta
Finished!
```

**Run create_core.sh**

This will generate Solr cores and index the data. Be sure to pass the name of your keyspace as a parameter:
```
./create_core.sh <name of your keyspace>
...
Creating Solr cores...
finished creating Solr cores!
```


The Amazon data model includes the following tables:


Click stream data:
```
CREATE TABLE <your keyspace name>.clicks (
    asin text,
    seq timeuuid,
    user uuid,
    area_code text,
    city text,
    country text,
    ip text,
    loc_id text,
    location text,
    location_0_coordinate double,
    location_1_coordinate double,
    metro_code text,
    postal_code text,
    region text,
    solr_query text,
    PRIMARY KEY (asin, seq, user)
) WITH CLUSTERING ORDER BY (seq DESC, user ASC);
```
And book metadata: 

```
CREATE TABLE <your keyspace name>.metadata (
    asin text PRIMARY KEY,
    also_bought set<text>,
    buy_after_viewing set<text>,
    categories set<text>,
    imurl text,
    price double,
    solr_query text,
    title text
);
```

So what are things you can do? 

First, set our ***default*** keyspace so that we dont need to type it in every time.

```
use <yourkeyspace>;
```

**Filter queries**: These are awesome because the result set gets cached in memory. 
```
SELECT * FROM metadata WHERE solr_query='{"q":"title:Noir~", "fq":"categories:Books", "sort":"title asc"}' limit 10; 
```

**Faceting**: Get counts of fields 
```
SELECT * FROM metadata WHERE solr_query='{"q":"title:Noir~", "facet":{"field":"categories"}}' limit 10; 
```

**Geospatial Searches**: Supports box and radius
```
SELECT * FROM clicks WHERE solr_query='{"q":"asin:*", "fq":"+{!geofilt pt=\"37.7484,-122.4156\" sfield=location d=1}"}' limit 10; 
```

**Joins**: Not your relational joins. These queries 'borrow' indexes from other tables to add filter logic. These are fast! 
```
SELECT * FROM metadata WHERE solr_query='{"q":"*:*", "fq":"{!join from=asin to=asin force=true fromIndex=<your keyspace name>.clicks}area_code:415"}' limit 5; 
```

**Fun all in one**
```
SELECT * FROM metadata WHERE solr_query='{"q":"*:*", "facet":{"field":"categories"}, "fq":"{!join from=asin to=asin force=true fromIndex=<your keyspace name>.clicks}area_code:415"}' limit 5;
```

Want to see a really cool example of a live DSE Search app? Check out [KillrVideo](http://www.killrvideo.com/) and its [Git](https://github.com/luketillman/killrvideo-csharp) to see it in action. 

----------


Hands On DSE Analytics
--------------------

Spark is general cluster compute engine. You can think of it in two pieces: **Streaming** and **Batch**. 
**Streaming** is the processing of incoming data (in micro batches) before it gets written to Cassandra (or any database). 
**Batch** includes both data crunching code and **SparkSQL**, a hive compliant SQL abstraction for **Batch** jobs. 

It's a little tricky to have an entire class run streaming operations on a single cluster, so if you're interested in dissecting a full scale streaming app, check out [THIS git](https://github.com/retroryan/SparkAtScale).  

>Spark has a REPL we can play in. But to make things easy, first we'll use the Spark SQL REPL:

```dse spark-sql --conf spark.ui.port=<Pick a random 4 digit number> --conf spark.cores.max=1```

>Notice the spark.ui.port flag - Because we are on a shared cluster, we need to specify a radom port so we don't clash with other users. We're also setting max cores = 1 or else one job will hog all the resources. 

Try some unfamiliar CQL commands on that Amazon data - like a sum on a column:

```
use <your keyspace name>.;
SELECT sum(price) FROM metadata;
```
You should see the following output:
```
140431.25000000006
```
Try a join on two tables:
```
SELECT m.title, c.city FROM metadata m JOIN clicks c ON m.asin=c.asin;
```
Output for example:
```
Major Legal Systems in the World Today: An Introduction to the Comparative Study of Law	San Francisco
Major Legal Systems in the World Today: An Introduction to the Comparative Study of Law	San Francisco
Major Legal Systems in the World Today: An Introduction to the Comparative Study of Law	San Francisco
...
Major Legal Systems in the World Today: An Introduction to the Comparative Study of Law	San Francisco
Major Legal Systems in the World Today: An Introduction to the Comparative Study of Law	South San Francisco
Major Legal Systems in the World Today: An Introduction to the Comparative Study of Law	San Francisco
```
Sums and groups:
```
SELECT asin, sum(price) AS max_price FROM metadata GROUP BY asin ORDER BY max_price DESC limit 1;
```
Output:
```
B0002GYI5A      899.0
```

Now let's try an excercise using the Spark REPL.
We will load a csv file into a Cassandra table.

In the repo directory:
Start cqlsh like this from the command prompt on one of the nodes in the cluster:

```
cqlsh <node private ip address>
``` 
or
```
cqlsh <node name>
```

Then :
```
USE <your keyspace name>;

CREATE TABLE albums (
    artist text,
    album text,
    year text,
    country text,
    quality text,
    status text,
    PRIMARY KEY ((artist,album, year,country))
);

COPY albums FROM 'albums.csv' WITH HEADER=TRUE;
```

To exit the cqlsh, type ```exit```

Now we start the Spark REPL :
```
dse spark --conf spark.cores.max=1
```
Once inde the REPL we can run some Scala commands:
```
println("Spark version:"+sc.version)
```
We need to import SQLContext so that we can create a SQLContext:
```
import org.apache.spark.sql.SQLContext
val sqlContext = new SQLContext(sc)
```
Now we can create a dataframe from our Cassandra table:
```
val df_albums = sqlContext.read.format("org.apache.spark.sql.cassandra").options(Map("keyspace" -> "<your keyspace name>", "table" -> "albums")).load().cache
```
>Dataframes were introduced in Spark 1.3 and are a more efficient means of managing and analysing data than traditional Spark RDD's - you can read more about them [here](https://databricks.com/blog/2015/02/17/introducing-dataframes-in-spark-for-large-scale-data-science.html) and a good explanation [here](http://stackoverflow.com/questions/31508083/difference-between-dataframe-and-rdd-in-spark) 

We can view the schema of a DataFrame:
```
scala> df_albums.printSchema()
root
 |-- artist: string (nullable = true)
 |-- album: string (nullable = true)
 |-- year: string (nullable = true)
 |-- country: string (nullable = true)
 |-- quality: string (nullable = true)
 |-- status: string (nullable = true)
```
We can display some of the records (by default the first 20):
```
scala> df_albums.show()
+------------------+--------------------+----+------------------+-------+--------------+
|            artist|               album|year|           country|quality|        status|
+------------------+--------------------+----+------------------+-------+--------------+
|      Miss Platnum|               Chefa|2007|           Germany| normal|      Official|
|       Mount Eerie|             I Whale|2005|               USA| normal|      Official|
|        Jerry Reed|         Me and Chet|1972|               USA| normal|      Official|
|Insane Clown Posse|         Hokus Pokus|1998|               USA| normal|     Promotion|
|            Deluxe|Colillas en el suelo|2007|             Spain| normal|      Official|
|    The Low Anthem|2009-06-18: Daytr...|2009|           Unknown| normal|     Promotion|
|Hardcore Superstar|Mother's Love / S...|  -1|           Unknown| normal|      Official|
|         Sub Focus|              Splash|2010|    United Kingdom| normal|      Official|
|  Benjamin Diamond|      Fit your heart|  -1|           Unknown| normal|      Official|
|              Kane|       Shot of a Gun|2008|       Netherlands| normal|      Official|
|            Tiësto|Magik One: First ...|2000|       Netherlands| normal|      Official|
|       Regenerator|     Everyone Follow|1994|               USA| normal|      Official|
|    Duke Ellington|Music by Ellingto...|1986|           Unknown| normal|      Official|
|   Mott the Hoople|      The Collection|1987|            France| normal|      Official|
|          R. Kelly|       Bump N' Grind|1994|               USA| normal|      Official|
|        Duvelduvel|        Puur Kultuur|2007|       Netherlands| normal|      Official|
|       StoneBridge|        Take Me Away|2005|    United Kingdom| normal|      Official|
|     Pig Destroyer|                Demo|1997|               USA| normal|Pseudo-Release|
|         Ленинград|Мат без электриче...|1999|Russian Federation| normal|      Official|
| Jacques Offenbach|         Pomme d'api|1983|            France| normal|      Official|
+------------------+--------------------+----+------------------+-------+--------------+
```
We can FILTER the results:
```
scala> df_albums.filter("year > 2000").show()
+--------------------+--------------------+----+--------------+-------+--------------+
|              artist|               album|year|       country|quality|        status|
+--------------------+--------------------+----+--------------+-------+--------------+
|        Miss Platnum|               Chefa|2007|       Germany| normal|      Official|
|         Mount Eerie|             I Whale|2005|           USA| normal|      Official|
|              Deluxe|Colillas en el suelo|2007|         Spain| normal|      Official|
|      The Low Anthem|2009-06-18: Daytr...|2009|       Unknown| normal|     Promotion|
|           Sub Focus|              Splash|2010|United Kingdom| normal|      Official|
|                Kane|       Shot of a Gun|2008|   Netherlands| normal|      Official|
|          Duvelduvel|        Puur Kultuur|2007|   Netherlands| normal|      Official|
|         StoneBridge|        Take Me Away|2005|United Kingdom| normal|      Official|
|    Jake Shimabukuro|   Play Loud Ukulele|2005|       Unknown| normal|      Official|
|               Foals|             Cassius|2008|United Kingdom| normal|      Official|
|               Clark|       Growls Garden|2009|United Kingdom| normal|      Official|
|        Constantines|   Too Slow for Love|2009|        Canada| normal|      Official|
|The Electric Soft...|   Holes in the Wall|2002|United Kingdom| normal|      Official|
|        Farben Lehre|           Pozytywka|2003|        Poland| normal|      Official|
|        Icon of Coil|III: The Soul Is ...|2006|       Unknown| normal|      Official|
|            DJ Marky|FabricLive 55: DJ...|2011|United Kingdom| normal|      Official|
|               中川幸太郎|Phantom -PHANTOM ...|2004|         Japan| normal|Pseudo-Release|
|         Jay Reatard|         In the Dark|2007|       Austria| normal|      Official|
|               Tryad|            The Tree|2011|       Unknown| normal|     Promotion|
|                Kiki|         Love Kills!|2012|       Germany| normal|      Official|
+--------------------+--------------------+----+--------------+-------+--------------+
```
We can GROUP BY and COUNT:
```
scala> df_albums.groupBy("year").count().show()
+----+-----+                                                                    
|year|count|
+----+-----+
|1917|   10|
|1918|    6|
|1919|    4|
|1980|  807|
|1981|  853|
|1982| 1047|
|1983|  975|
|1984| 1007|
|1985|  998|
|1986| 1081|
|1987| 1288|
|1988| 1323|
|1989| 1505|
|1920|    4|
|1921|    1|
|1922|    5|
|1923|    4|
|1924|   18|
|1925|    6|
|1926|    9|
+----+-----+
```
We can use the DataFrame to create an in-memory Spark SQL table:
```
df_albums.registerTempTable("spark_albums_table")

sqlContext.sql("SELECT * FROM spark_albums_table").show
+------------------+--------------------+----+------------------+-------+--------------+
|            artist|               album|year|           country|quality|        status|
+------------------+--------------------+----+------------------+-------+--------------+
|      Miss Platnum|               Chefa|2007|           Germany| normal|      Official|
|       Mount Eerie|             I Whale|2005|               USA| normal|      Official|
|        Jerry Reed|         Me and Chet|1972|               USA| normal|      Official|
|Insane Clown Posse|         Hokus Pokus|1998|               USA| normal|     Promotion|
|            Deluxe|Colillas en el suelo|2007|             Spain| normal|      Official|
|    The Low Anthem|2009-06-18: Daytr...|2009|           Unknown| normal|     Promotion|
|Hardcore Superstar|Mother's Love / S...|  -1|           Unknown| normal|      Official|
|         Sub Focus|              Splash|2010|    United Kingdom| normal|      Official|
|  Benjamin Diamond|      Fit your heart|  -1|           Unknown| normal|      Official|
|              Kane|       Shot of a Gun|2008|       Netherlands| normal|      Official|
|            Tiësto|Magik One: First ...|2000|       Netherlands| normal|      Official|
|       Regenerator|     Everyone Follow|1994|               USA| normal|      Official|
|    Duke Ellington|Music by Ellingto...|1986|           Unknown| normal|      Official|
|   Mott the Hoople|      The Collection|1987|            France| normal|      Official|
|          R. Kelly|       Bump N' Grind|1994|               USA| normal|      Official|
|        Duvelduvel|        Puur Kultuur|2007|       Netherlands| normal|      Official|
|       StoneBridge|        Take Me Away|2005|    United Kingdom| normal|      Official|
|     Pig Destroyer|                Demo|1997|               USA| normal|Pseudo-Release|
|         Ленинград|Мат без электриче...|1999|Russian Federation| normal|      Official|
| Jacques Offenbach|         Pomme d'api|1983|            France| normal|      Official|
+------------------+--------------------+----+------------------+-------+--------------+
```

And now we can perform complex SQL operations on the data in Spark memory:
```
scala> sqlContext.sql("SELECT country,count(*) as nb FROM spark_albums_table group by country having count(*)>=200 order by nb").show
+------------------+----+                                                       
|           country|  nb|
+------------------+----+
|       Switzerland| 204|
|           Estonia| 207|
|          Portugal| 217|
|           Jamaica| 234|
|            Mexico| 262|
|           Austria| 318|
|           Denmark| 345|
|         Argentina| 523|
|            Turkey| 617|
|Russian Federation| 743|
|           Belgium| 757|
|            Brazil| 865|
|            Norway| 887|
|            Poland|1108|
|             Spain|1344|
|       Netherlands|1395|
|         Australia|1541|
|             Italy|1633|
|            Canada|1809|
|           Finland|1966|
+------------------+----+
```
To exit the REPL type ```exit```

You can see a great demo of running thse steps inside the Zeppelin Notebook [here](https://github.com/victorcouste/zeppelin-spark-cassandra-demo/)


----------


DSE Streaming Demo
--------------------
**Spark Notebook**

Much like Zeppelin, [Spark Notebook](http://spark-notebook.io/) is an awesome tool for exploring Spark and making simple visualizations. It's not a DataStax product - but for a great demo from another of our DataStax colleagues see: https://github.com/retroryan/twitter_classifier/tree/master/notebooks/TweetAnalysis

>Have fun with it! See what you come up with :)

----------


Getting Started With DSE Ops
--------------------

Most of us love to have tools to monitor and automate database operations. For Cassandra, that tool is DataStax OpsCenter. If you prefer to roll with the command line, then two core utilities you'll need to understand are nodetool and dsetool.

**Utilities you'll want to know:**
```
nodetool  //Cassandra's main utility tool
dsetool   //DSE's main utility tool
```
**nodetool Examples:**
```
nodetool status  //shows current status of the cluster 

nodetool tpstats //shows thread pool status - critical for ops
```

**dsetool Examples:**
We've previously used the dsetool command to create Solr cores. We can also use it to obtain a cluster status report:
```
dsetool status //shows current status of cluster, including DSE features
```

**The main log you'll be taking a look at for troubleshooting outside of OpsCenter:**
```
/var/log/cassandra/system.log
```

To see OpsCenter UI, open the browser to
```
 http://<your cluster IP>:8888
```

