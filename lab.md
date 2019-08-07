# KSQL Workshop

## Pre-reqs / testing the setup

#### open up a terminal session to the gateway server
> (for MacOS users) `ssh <username>@ec2-52-27-37-214.us-west-2.compute.amazonaws.com`
>
> (for Windows users) `putty <username>@ec2-52-27-37-214.us-west-2.compute.amazonaws.com`

where `<username>` is your first initial plus last name. You will be prompted for a password, which is `ksqlr0ck$`

#### test that you can open web the control center web-page
open up a browser window and navigate to `http://controlcenter-<username>.selabs.net:9021`
the login credentials for the web-page are `admin/Developer1`

### In the beginning, there was the CLI
We will mostly be using the KSQL CLI tool to interact with the KSQL Server. If you've used a tool like MySQL before this should be very familiar.
Let's fire it up!

`ksql`

Admire the ASCII-art :-)

### Looking around
Let's quickly get familiar with this environment and take a look around:
* `show topics;`
* `show streams;`
TODO navigating command history etc.....
`print 'xxxx' from beginning;`
`print 'xxxx' limit 5;`

## Create some data
1. write json records to file

1. mysql connector

`run stuff`

1. datagen connector - (explain!)

`run something else`

1. back over in KSQL, let's check out our new topics and the data inside
```
print stuff....
```

### Getting started with DDL
To make use of these topics in KSQL we need to define some Streams and/or Tables over them
create stream customersraw with(kafka_topic='rds.<username>.CUSTOMERS-raw', value_format='AVRO');
1. declare stream over json file

`create .....`

1. describe it
```
describe foo;
```
1. now test it!
```sql
select * from ......
```
1. Register the RATINGS data as a KSQL stream, sourced from the ratings topic
```sql
create stream ratings with (kafka_topic='ratings', value_format='avro');`
```
1. now query from this new stream. what happens ? and why ?
we'll come back and fix this in a second...

`ksql> create stream cust_cdc with(kafka_topic='dbservername.demo.CUSTOMERS', value_format='AVRO');`

-- Re-partition on the ID column and set the target topic to
-- match the same number of partitings as the source ratings topic:
`CREATE STREAM CUSTOMERS_SRC_REKEY WITH (PARTITIONS=2) AS SELECT * FROM CUSTOMERS_SRC PARTITION BY ID;`
```
create stream test as select after->id as id, after->first_name as first_name, after->last_name as last_name, after->email as email, after->club_status as club_status, after->comments as comments from xxxxx partition by id;
```
-- Register the CUSTOMER data as a KSQL table, sourced from the re-partitioned topic

`CREATE TABLE CUSTOMERS WITH (KAFKA_TOPIC='CUSTOMERS_SRC_REKEY', VALUE_FORMAT ='AVRO', KEY='ID');`


```
set 'auto.offset.reset' = 'earliest';
```

## Changing data in MySQL
In a new terminal window, launch the MySQL client
```bash
mysql
```
You should be able to see your source CUSTOMERS table here, and inspect it's records with `select * from CUSTOMERS` (note the table name is case-sensitive!)
Try inserting a new record or updating an existing one
> Example: update name of a record to be your own name
> `update CUSTOMERS set first_name = 'janet', last_name='smith' where id = 1;`


## so what's actually happening here ?
```
show queries;
explain <query_id>;  (case sensitive!)
```

create stream test4 as select r.rowkey, r.user_id, c.rowkey, c.first_name, r.stars, r.channel, r.route_id, c.club_status
create stream test4 as select r.user_id, c.first_name, r.stars, r.channel, r.route_id, c.club_status
>from ratings r
>left join customers c
>on r.user_id = c.rowkey;


select * from test4 where stars < 3 and channel like 'iOS%' and lcase(club_status) = 'platinum) limit 5;

c3 - consumers view - pick a join one - 

## here be dragons
* -- Perform the join, writing to a new topic.
CREATE STREAM RATINGS_WITH_CUSTOMER_DATA \
       WITH (PARTITIONS=1, \
             KAFKA_TOPIC='ratings-enriched') \
       AS \
SELECT R.RATING_ID, R.MESSAGE, R.STARS, R.CHANNEL,\
      C.ID, C.FIRST_NAME + ' ' + C.LAST_NAME AS FULL_NAME, \
      C.CLUB_STATUS, C.EMAIL \
      FROM RATINGS R \
        LEFT JOIN CUSTOMERS C \
        ON R.USER_ID = C.ID \
      WHERE C.FIRST_NAME IS NOT NULL;


CREATE STREAM UNHAPPY_PLATINUM_CUSTOMERS \
       WITH (VALUE_FORMAT='JSON', PARTITIONS=1) AS \
SELECT FULL_NAME, CLUB_STATUS, EMAIL, STARS, MESSAGE \
FROM   RATINGS_WITH_CUSTOMER_DATA \
WHERE  STARS < 3 \
  AND  CLUB_STATUS = 'platinum';
  
  
	4- Extract dbz change and re-key
	5- Join ratings to users
	6- Filter ratings by score and user status
	7- Aggregate filtered ratings by 5 mins window
	8- Where to join the insert-able topic ?
	10- How to work in a CASE stmt ?
	11- Show processing log ?
	12- Create stream on cmd topic ?? Just for info, don't mess with this! :)
	13- Show testing tool example

*
*

