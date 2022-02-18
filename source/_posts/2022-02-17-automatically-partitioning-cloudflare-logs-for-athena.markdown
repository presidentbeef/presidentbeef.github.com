---
layout: post
title: "Automatically Partitioning Cloudflare Logs for Athena"
date: 2022-02-17 21:00
comments: true
categories: 
---

If you are using Cloudflare, it can be helpful to [configure Cloudflare to push request logs to S3](https://developers.cloudflare.com/logs/get-started/enable-destinations/aws-s3). Otherwise, the Cloudflare dashboard provides only a limited view into your data (72 hours at a time and sampled data instead of full logs).

Once the Cloudflare request logs are in S3, they can be queried using [Athena](https://aws.amazon.com/athena/). [This blog post](https://duffn.medium.com/analyzing-cloudflare-logs-with-aws-athena-19c1399c91e3) even provides a nice `CREATE TABLE` command to set up the table in Athena.

However, there is a problem. When performing a query in Athena, it _might_ have to scan _all_ of the logs in S3, even if you try to limit the query. This can be slow and costly, as Athena queries are charged per byte scanned.

The only way to really limit the amount of data scanned is to [partition the data](https://docs.aws.amazon.com/athena/latest/ug/partitions.html).

This post assumes you have already set up Cloudflare to push logs to an S3 bucket, configured a database in Athena to access it, and then realized those logs will grow forever, along with your query times.

(If you just want the "how to" without the exposition, jump down to "Setting Up Partitions for Cloudflare Logs".)

### Partitioning

Most commonly, you will want to look at logs from a specific time period, so it makes sense to partition the logs by date.

Most of the partitioning documentation suggests the files (or in this case, S3 objects) include a `column=value` key pair in the name. The `column` can then be used as a partition.

Unfortunately, Cloudflare does not allow customizing the format of the file names it produces.

_Fortunately_, the file names do include date/time information. The logs are grouped by date and time range:

```
s3://mah_s3_bucket/20210812/20210812T223000Z_20210812T224000Z_9af500e2.log.gz
```

So all we need to do is grab that date "folder" name and that's our partition! Easy, right?

No, wrong. This is AWS. _Nothing is easy._

### Use a Recurring Job?

Several of the AWS documentation pages suggest using `ALTER TABLE` to [`ADD PARTITION`s](https://docs.aws.amazon.com/athena/latest/ug/alter-table-add-partition.html).

Something like this:

```sql
ALTER TABLE cloudflare_logs ADD
  PARTITION (dt = '2021-08-12') LOCATION 's3://mah_log_bucket/20210812/';
```

But since the logs will grow every day, we'll need to add new a new partition every 24 hours. It is not possible to "predefine" the partitions.

This requires setting up a recurring job... somewhere... to periodically define the new partitions. So now we have to pull in _another_ AWS service to make S3 and Athena work nicely?! No thanks!

### Partition Projection

At the bottom of the page about partitions, there is a paragraph about "[partition projection](https://docs.aws.amazon.com/athena/latest/ug/partitions.html#partitions-partition-projection)" that sounds promising:

> To avoid having to manage partitions, you can use partition projection. Partition projection is an option for highly partitioned tables whose structure is known in advance.

Yes, this is what we want! But how does it work?

Essentially like this:

We must define a column, its type, start and end values, and the interval between those values. Athena will then be able to extrapolate all the possible values.

Then we define a pattern to pull the value out of each S3 object _name_. This allows Athena to figure out which objects (logs) are associated with which partition value.

For example, the partition `20210812` will be associated with `s3://mah_s3_bucket/20210812/20210812T223000Z_20210812T224000Z_9af500e2.log.gz`

Once that's all done, we can query based on the partition as if it were a column, like:

```sql
SELECT * FROM cloudflare_logs
WHERE log_date >= '20210812'
  AND log_date < '20210901';
```

## Setting Up Partitions for Cloudflare Logs

Here are the steps that _must_ be taken to set up the partitions:

1. Add the partition "column" when creating the table
2. Set several properties on the table to define the projection
3. Set the partition pattern to match against object names
4. Enable projection

(Actually, 2-4 are all the same: set (totally unvalidated) key-value properties on the table.)

Fortunately, all of this can be accomplished with one giant Athena command:

```sql
CREATE EXTERNAL TABLE `YOUR_TABLE_NAME`(
  `botscore` int,
  `botscoresrc` string,
  `cachecachestatus` string,
  `cacheresponsebytes` int,
  `cacheresponsestatus` int,
  `clientasn` int,
  `clientcountry` string,
  `clientdevicetype` string,
  `clientip` string,
  `clientipclass` string,
  `clientrequestbytes` int,
  `clientrequesthost` string,
  `clientrequestmethod` string,
  `clientrequestpath` string,
  `clientrequestprotocol` string,
  `clientrequestreferer` string,
  `clientrequesturi` string,
  `clientrequestuseragent` string,
  `clientsslcipher` string,
  `clientsslprotocol` string,
  `clientsrcport` int,
  `edgecolocode` string,
  `edgecoloid` int,
  `edgeendtimestamp` string,
  `edgepathingop` string,
  `edgepathingsrc` string,
  `edgepathingstatus` string,
  `edgeratelimitaction` string,
  `edgeratelimitid` int,
  `edgerequesthost` string,
  `edgeresponsebytes` int,
  `edgeresponsecontenttype` string,
  `edgeresponsestatus` int,
  `edgeserverip` string,
  `edgestarttimestamp` string,
  `firewallmatchesactions` array<string>,
  `firewallmatchesruleids` array<string>,
  `firewallmatchessources` array<string>,
  `originip` string,
  `originresponsestatus` int,
  `originresponsetime` int,
  `originsslprotocol` string,
  `rayid` string,
  `wafaction` string,
  `wafflags` string,
  `wafmatchedvar` string,
  `wafprofile` string,
  `wafruleid` string,
  `wafrulemessage` string,
  `workersubrequest` boolean,
  `zoneid` bigint)
PARTITIONED BY (
  `YOUR_COLUMN_NAME` string)
ROW FORMAT SERDE
  'org.openx.data.jsonserde.JsonSerDe'
STORED AS INPUTFORMAT
  'org.apache.hadoop.mapred.TextInputFormat'
OUTPUTFORMAT
  'org.apache.hadoop.hive.ql.io.IgnoreKeyTextOutputFormat'
LOCATION
  's3://YOUR_BUCKET_NAME/'
TBLPROPERTIES (
  'projection.enabled'='TRUE',
  'projection.YOUR_COLUMN_NAME.format'='yyyyMMdd',
  'projection.YOUR_COLUMN_NAME.interval'='1',
  'projection.YOUR_COLUMN_NAME.interval.unit'='DAYS',
  'projection.YOUR_COLUMN_NAME.range'='YOUR_START_DATE,NOW',
  'projection.YOUR_COLUMN_NAME.type'='date', 
 
 'storage.location.template'='s3://YOUR_BUCKET_NAME/${YOUR_COLUMN_NAME}/'
) 
```

These are the pieces related to partitioning:

```sql
PARTITIONED BY (
  `YOUR_COLUMN_NAME` string)
```

This tells Athena to set up a column for the partition.

(The type of the partition column **must** be `string`, even though the projection type **must** be `date`. Does this make any sense? **NO**.)

Then the table properties.

Turn on projection:

```sql
'projection.enabled'='TRUE'
```

Set the column type to date:

```sql
'projection.YOUR_COLUMN_NAME.type'='date'
```

This is so Athena knows how to interpolate values.

Define the date format to match:

```sql
'projection.YOUR_COLUMN_NAME.format'='yyyyMMdd'
```

Set the interval for the values to one day:

```sql
'projection.YOUR_COLUMN_NAME.interval'='1',
'projection.YOUR_COLUMN_NAME.interval.unit'='DAYS'
```

Set the range for the values:

```sql
'projection.YOUR_COLUMN_NAME.range'='YOUR_START_DATE,NOW'
```
 
`NOW` is a special value so the end of the range will always be the current day.

This sets a template to extract the date string from the object name, using the date template defined above, and setting the value in the column name specified for the projection: 

```sql
'storage.location.template'='s3://YOUR_BUCKET_NAME/${YOUR_COLUMN_NAME}/'
```

If you are unfamiliar with Athena, it's good to know that deleting/creating tables is low impact. If the table is already created, it is not a big deal to delete it and start over.

Here are the important bits above that you will need to change:

* `YOUR_TABLE_NAME` is whatever you want to name the table. Something like `cloudflare_logs` would probably make sense.
* `YOUR_COLUMN_NAME` is whatever you want to name the projection "column". Could be `dt` like in the AWS docs, or `log_date` or whatever you want.
* `YOUR_BUCKET_NAME` is the name of the S3 bucket.
* `YOUR START_DATE` is the date of the first log. Something like `20210101`.


The table should now indicate it is partitioned:

![Table name with text 'partitioned' next to it](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/z17sufwwkndnfbygh7xm.png)

And the partition should show up as a column:

![Column name with text 'string (partitioned)' next to it](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/2k9ixdqd6820stqrvetr.png)

### Using Date Partitions

To test if partitions are working as expected, a quick query like this will work:

```sql
SELECT DISTINCT(YOUR_COLUMN_NAME)
FROM "YOUR_TABLE_NAME"
LIMIT 10;
```

The expected output is several dates for which there are logs.

Once that is confirmed, the partition column can be used like any other column. Since the values is a basic ISO date format, comparison operators can be safely used even though the column is really just a string.

For a single day:

```sql
SELECT * FROM YOUR_TABLE
WHERE YOUR_COLUMN_NAME = "20200101";
```

For an inclusive range:

```sql
SELECT * FROM YOUR_TABLE
WHERE YOUR_COLUMN_NAME >= "20200101"
  AND YOUR_COLUMN_NAME <= "20210101";
```

And now your queries can be faster and cheaper!
