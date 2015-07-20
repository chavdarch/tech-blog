---
layout: post
title: Introducing Calatrava
author: Chavdar Chernashki
date: '2015-07-21'
categories: 'tech blog'
tags:
- stream
- event
- database
---

<p align="center">
  <img src="http://i.imgur.com/gLphNAo.jpg"/>
</p>


##Calatrava - a service to publish and manage streams of database events

###Problem

Lots of applications need to react quickly to changes on the underlying data.
For example, it would be great if we can quickly synchronize a change to the name of a product in the database with the search for that name through the website. 

In order to properly react to a data change we need to be able to:

* **listen** to a predefined data source for changes
* **publish** each change in a specific format to a well known location
* **notify** all interested  consumers of those changes

**Calatrava** is a solution for the first two (listen and publish) parts of the problem. 

###How it works
<p align="center">
  <img src="http://i.imgur.com/ciOFtXt.png"/>
</p>


1. **data change** - can come from many different places (ui, cron jobs, batch imports, etc.)
      Data changes can happen simultaneously across:
   * separate DB tables in the database
   * separate rows of the same DB table
   
2. **change event fired** -  change event is: 
   * triggered immediately  after   successful commit   of the corresponding DB transaction that is changing the data.
   * one change event for each DB row change(update,delete, create) for each table, we are interested in.
   
3.  **change event storing** - 
We have a log table, containing change events for each source table.
For example brands_log for brands table.
Storing a change event before publishing it would prevent loss in case of a publishing component failure(restart).

4. **retrieving change events** - 
	* There is a separate log table per source table.
	* log entries for any log table are fetched and processed sequentially by a single actor. 

5. **publishing change events** - 
    * change publisher would publish each change event, immediately after retrieving it, to a dedicated stream for this event.
    * Having one(e.g brand , sku, product, etc.. ) stream per event would enable us to stream all types(sku, product, ...) change events in parallel  to  fine tune publishing per type (e.g we could decide to publish events from specific type in a batch , or create additional shards(if we use Kinesis stream) for specific stream )           

6. **consuming change events** - 
	* each stream provides durable storage for each event type for specific amount  of time 
    * each consumer is able to read events from each stream in parallel
    * each consumer receives a change event at most 20 seconds after event was generated
    * the order of the events in each stream is chronological 
    * data format of change events is json. 

###How to setup

Calatrava requries special permissions to your database and your AWS sink, so that it can read records from the former, and publish events to the latter.

##### Database Access

The best way to set up for Calatrava is to prepare a separate user and schema. The user should be granted all privileges on the new schema.

Calatrava service will provide the SQL code needed to **create a log table** in the new schema **and the triggers** that will monitor the source table and inject data into the log table when the source table is modified.

Finally, the **database** to be monitored **must be accessible** from Calatrava API service. 

##### AWS Sink Access

Right now, the **only supported data sink** is based on AWS services. You should create a **Kinesis** stream for each source table, and Calatrava will publish events on this stream whenever the source table is modified.

Due to data size limitations, **large events** (more than 50Kb) cannot be sent through Kinesis. Instead, Calatrava writes the event data as an object in **S3**, and sends the object key through Kinesis.

The wrapper for a change in the source table is `ChangeEvent`, and its structure is defined in api.json as follows:

Field | Type | Description | Notes
:---- | :--- | :---------- | :----
id | String | Identifier for this change event | Mandatory
entity_key | String | The key of the modified source table entity | Mandatory
before_json | String | JSON representation of the source table entity before the change | Optional
after_json | String | JSON representation of the source table entity after the change | Optional
timestamp | DateTime | ISO-8601 representation of the change timestamp | Mandatory

If the entity is inserted into the source table, then `before_json` is going to be null. If the entity is deleted, then `after_json` is going to be null. Both values will be sent for an updated entity.

NOTE: for schema-evolution-manager tables, the rows are never deleted, but updated with `deletedBy` and `deletedAt` fields set.

The structure for events sent through Kinesis is called `SinkEvent`, and the definition follows:

Field | Type | Description | Notes
:---- | :--- | :---------- | :----
event | `ChangeEvent` | The actual change event, if size less than 50Kb | Optional
event_object_key | String | The object key for the event data in S3 | Optional

For large events, data will be written in S3, and the `SinkEvent` will contain no value for `event`, but will contain the `event_object_key`. For smaller events, the data will be contained in the `event` field, while the `event_object_key` will be null.

To create an **AWS Sink** appropriate for Calatrava, you need to create a **Kinesis stream**, and an **S3 bucket**. Allocate sufficient shards in the Kinesis stream to suit your expected data volumes.

Next, create an **IAM role** that Calatrava can assume to gain **access to the stream and bucket**.

Set up a **policy** for this role that will have the appropriate permissions. At the minimum, it should have permission to **PutRecord** into the **Kinesis stream**, and **full permissions** for the **S3 bucket**. Here's an example policy which gives permission to put records in all streams whose names start with `updates` and all permissions to a bucket named `for-calatrava`:

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "Stmt1427710471000",
            "Effect": "Allow",
            "Action": [
                "kinesis:PutRecord"
            ],
            "Resource": [
                "arn:aws:kinesis:us-east-1:698112785971:stream/updates*"
            ]
        },
        {
            "Sid": "Stmt1427710592000",
            "Effect": "Allow",
            "Action": [
                "s3:*"
            ],
            "Resource": [
                "arn:aws:s3:::for-calatrava",
                "arn:aws:s3:::for-calatrava/*"
            ]
        }
    ]
}
```

Finally, the **role** must have a **trust relationship** to the AWS **account** where **Calatrava** runs, so that Calatrava can assume this role. This can be achieved by adding this statement to the trust policy:

```
"Statement": [
    {
      "Sid": "1",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::856716094854:root"
      },
      "Action": "sts:AssumeRole"
    }
```

That account ID is the one where Calatrava service is running.

### How to use

Calatrava consists of two modules:

-  **a backend API service**, which provides a RESTful interface for interacting with Calatrava. This module also contains the actors that monitor database tables and publish change events to AWS Kinesis streams.

- **a frontend UI service**, which uses the REST API for configuration and monitoring of the running calatrava service 

You can use the UI service for the following:

 1. **singup** - 
Before using Calatrava, one must sign up for the service, using a valid email address. After confirming the email address, a user's account is automatically activated.

 2. **create an organization** - 
The top level grouping unit is the **organization**. The organization acts as the umbrella for grouping users and their databases. Any user can create a new organization with unique ID within Calatrava. 

 3. **add members to organization** - 
After creating the organization, the user who created it is the sole member. Any organization member can add more users or remove users from the organization. Only members of a given organization can monitor the databases, defined for the prganization. 
 
 4. **create databases for the organization** - 
For each organization, one or more **databases** can be created. Each table that need to be monitored is part of a database. 

 5. **create brides for each database** - 
You need to **define** a bridge (a connection from a database table to a stream) through the UI service for each interested table. After making sure that the prerequisites(db and AWS permissions) described above are in place you can **activate** the bridge through the UI. After that every INSERT, UPDATE or DELETE happening in the source table will appear as an AWS Kinesis event that carries a before and an after field in JSON form.

### Alternatives

*  **Linkedin databus**

Databus is using database `log mining` approach , where database is considered the single source-of-truth and all changes are extracted from its transaction or commit log.

This approach gives data consistency guarantee, but is practically hard to implement because different databases have transaction log formats and replication solutions that are proprietary and not guaranteed to have stable on-disk or on-the-wire representations across version upgrades.

Databus priovides data source agnostic solution avoiding technology lock-in and tie-in to binary formats throughout the application stack.

[https://github.com/linkedin/databus](https://github.com/linkedin/databus)

*  **Facebook Wormhole**

Wormhole is a publish-subscribe (pub-sub) system developed for use within Facebook’s geographically replicated datacenters. It is used to reliably replicate changes
among several Facebook services including TAO, Graph Search and Memcache.

[https://www.usenix.org/system/files/conference/nsdi15/nsdi15-paper-sharma.pdf](https://www.usenix.org/system/files/conference/nsdi15/nsdi15-paper-sharma.pdf)

 *  **Postgres 9.4 Logical Decoding** - 
 
PostgreSQL 9.4 provides infrastructure to stream the modifications performed via SQL to external consumers. Changes are sent out in streams identified by logical replication slots. 
The format in which those changes are streamed is determined by the output plugin used.

Every output plugin has access to each individual new row produced by INSERT and the new row version created by UPDATE. Availability of old row versions for UPDATE and DELETE depends on the configured replica identity.

Changes can be consumed either using the streaming replication protocol ( or by calling functions via SQL.
 
[http://www.postgresql.org/docs/9.4/static/logicaldecoding.html](http://www.postgresql.org/docs/9.4/static/logicaldecoding.html)

* **Apache Samza**

Below is an article describing how to use Apache Samza to turn the database inside-out, by representing each database table change relfected in the commit log as a stream event.

[http://www.confluent.io/blog/turning-the-database-inside-out-with-apache-samza/](http://www.confluent.io/blog/turning-the-database-inside-out-with-apache-samza/)

### Resource URL
**[https://github.com/gilt/calatrava](https://github.com/gilt/calatrava)**
