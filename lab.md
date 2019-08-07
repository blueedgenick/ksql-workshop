# KSQL Workshop

## Pre-reqs / testing the setup

#### open up a terminal session to the gateway server
> (for MacOS users) `ssh <username>@ec2-52-27-37-214.us-west-2.compute.amazonaws.com`
>
> (for Windows users) `putty <username>@ec2-52-27-37-214.us-west-2.compute.amazonaws.com`

where `<username>` is your first initial plus last name. You will be prompted for a password, which is `ksqlr0ck$`

#### test that you can open the control center web page
open up a browser window and navigate to `http://controlcenter-<username>.selabs.net:9021`
the login credentials for the web-page are `admin/Developer1`

Keep a copy of the KSQL syntax guide open in another browser tab: [https://docs.confluent.io/current/ksql/docs/developer-guide/syntax-reference.html]

### In the beginning, there was the CLI
We will mostly be using the KSQL CLI tool to interact with the KSQL Server. If you've used a tool like MySQL before this should be very familiar.
Let's fire it up!

`ksql`

Admire the ASCII-art :-)

### Looking around
Let's quickly get familiar with this environment by taking a look around:
* `show topics;`
* `show streams;`
You can navigate your KSQL command history much like a BASH shell:
* type `history` to see a list of previous commands
* `!123` will retrieve a previous command
* `ctrl-r` invokes a 'backward search' for commands matching whatever you type next, use arrow keys to navigate matches

Investigate session properties with `show properties;`

Investigate some data:
* `print 'xxxx' limit 3` or `print 'xxxx' from beginning limit 3;`

The topics we will use today are *`ratings`* and *`rds.<username>.CUSTOMERS-raw`*



## Getting started with DDL
To make use of these topics in KSQL we need to define some Streams and/or Tables over them

1. Register the RATINGS data as a KSQL stream, sourced from the ratings topic
```sql
create stream ratings with (kafka_topic='ratings', value_format='avro');`
```
2. Check your creation with `describe ratings;` and a couple of `select` queries. what happens ? why ?
  * try `describe extended ratings;`

3. Defining a lookup table for Customer data is a multi-step process from our CDC data-feed:
  ```
  create stream customers_cdc with(kafka_topic='rds.<username>.CUSTOMERS-raw', value_format='AVRO');
  ```
  * quickly query a couple of records to check it. Practice the art of "struct-dereferencing" with the "`->`" operator.
  
  * We want to extract just the changed record values from the CDC structures, re-partition on the ID column, and set the target topic to have the same number of partitings as the source `ratings` topic:
  ```
  create stream customers_flat with (partitions=1) as select after->id as id, after->first_name as first_name, after->last_name as last_name, after->email as email, after->club_status as club_status, after->comments as comments from customers_cdc partition by id;
  ```
  * Register the CUSTOMER data as a KSQL table, sourced from the re-partitioned topic
  ```
  create table customers with (kafka_topic='CUSTOMERS_FLAT', value_format='AVRO');
  ```
  
now query from this new stream. what happens ? and why ?
we'll come back and fix this in a second...
(
```
set 'auto.offset.reset' = 'earliest';
```
)

### Changing data in MySQL
In a new terminal window, launch the MySQL client
```bash
mysql
```
You should be able to see your source CUSTOMERS table here, and inspect it's records with `select * from CUSTOMERS` (note the table name is case-sensitive!)
Try inserting a new record or updating an existing one
> Example: update name of a record to be your own name
> `update CUSTOMERS set first_name = 'janet', last_name='smith' where id = 1;`

**TIP** if you leave your KSQL `select...from customers;` query running in the first window, watch what happens as you change data in the MySQL source


## Identify the unhappy customers

1.Back in KSQL, we start by finding just the low-scoring ratings
```
select * from ratings where stars < 3 and channel like 'iOS%' limit 5;
```
(play around with the `where` clause conditions to experiment)

Now convert this test query into a persistent one:
```
create stream poor_ratings as select * from ratings where stars < 3 and channel like 'iOS%';
```
2. Which of these low-score ratings was posted by an elite customer ? To answer this we need to join our customers table:
```
create stream poor_ratings_with_users as 
select r.user_id, c.first_name, c.last_name, c.club_status, r.stars, r.channel, r.route_id
from ratings r
left join customers c
on r.user_id = c.rowkey
where c.first_name is not null;
```
### ASIDE - so what's actually happening here ?
```
show queries;
explain <query_id>;  (case sensitive!)
```
over in the Control Center browser window, check out the consumer group for this join query, what do we see ? why ?


3. And we can then select from this new stream with a filter (where clause condition) on the `club_status` column:
```
create stream vip_poor_ratings as select * from poor_ratings_with_users where lcase(club_status) = 'platinum';
```
Note that we could have combined this filter clause into the previous query, eliminating the need for an extra step here.




  
## Extra Credit

Time permitting, let's explore the following ideas:
* mask the actual user names in the ouput
* aggregation - what if we just wanted to find the _really_ unhappy customers, who post multiple negative ratings in a short space of time ?
* explore and describe the available functions
* create a new stream over a topic that doesn't exist yet
* use `insert...values` to write a couple of test records into this new topic
* join it to one of our existing streams or tables

## Discussion Points
1. UDFs
1. Testing tools


