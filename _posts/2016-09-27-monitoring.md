---
title: System monitoring with InfluxDB vs Elasticsearch
teaser: Compare two modern solutions for monitoring
category: [Database]
tags: [Elasticsearch]
---

Monitoring computer systems had been alway important. It helped us know the system’s health, identify problems, and even to forecast them. 

Today, monitoring has become more and more significant because of several reasons:
- The cloud (or cheap memory) enables us to monitor and archive almost anything forever.
- The systems had become much more complicated, consists of a lot of moving parts such as VM’s, dockers, databases, services, you name it.
- IoT had come into our life with large amount of devices that we would like to track.
- Data analysis had evolved, enable us to get much more that only taking care of the system stability and health, it can help us understand our customers needs, identify the trends, classify our customers and more. 

In this blog post I will compare two modern solutions for monitoring. They have different approaches and implementation, and they have their advantages and disadvantages versus the other. The first is InfluxDB which is part of the TICK stack, and the second is Elasticsearch which is part of the ELK stack.

## What is monitoring?

Monitoring consists of the following activities:
- Collecting the data.
- Store the data.
- Visualize the data.
- Raise Alerts.

## TICK
TICK is:
- __Telegraf__ - collecting metrics and data on the system it’s running on or from other services.
- __InfluxDB__ - time series database with high availability and performance.
- __Chronograf__ - web application for data visualization.
- __Kapacitor__ - alerting and data processing engine.

The heart of the stack is InfluxDB which is a time series database.
Time series database (TSDB) is a software system that is optimized for handling time series data, arrays of numbers indexed by time. For instance, temperature of the water in the boiler over time, or CPU usage over time. TSDB is a NoSQL database that supports CRUD operation and queries. The main distinguish from other types of databases is it's optimization to maintain time indexes on a very large amount of records. 
Other leading TSDBs are _Graphite_, _Prometheus_, _OpenTSDB_ and more. The reason I chose InfluxDB is the fact that this is a modern database, written in GO, Very easy to setup and configure, and with great performance.

### InfluxDB data model
InfluxDB is schema less. You can add series, measurements and tags at any time. An example of a row:
```
app_degrees, country=Canada, city=Toronto degree=77.5 1422568543702900257
```

- Series - Collection of data that share a measurement and tag set (app_degrees).
- Tag - Key/value pair (country=Canada,  city=Toronto) - is optional, used to describe the measurement.
- Measurement - a numeric value ( degree=77.5)
- timestamp - the exact moment of the measurement (1422568543702900257).

This model enable to insert measurements efficiently and conveniently.

### Reading and writing into InfluxDB 

InfluxDB has an HTTP API and client libraries in many languages such as Ruby, GO, JAVA, Node and more.
InfluxDB enables writing single records using the API or inserting multiple rows from a file.

InfluxDB has a query language that is very similar to SQL. An example for a query:
```
select degrees from app_degrees where country = ‘Canada’
```
It has the following statement: SELECT, WHERE, GROUP BY, ORDER BY, LIMIT and more. A developer with an SQL knowledge will feel very convenient.

The results are in JSON format.

### Data visualization

“TICK” stack comes with Chronograf which is a web application for visualizing the series. It allows building graphs, tables, dashboards and more. It has a very understandable UX and you can build a dashboard in minutes.


### Other features of the stack
- Retention policy - you can configure when if ever the data will be deleted.
- Continues queries, that automatically compute aggregation on the data.
- High availability.

## ELK
ELK is:
- Elasticsearch - a search engine based on Lucene.
- Logstash - data collection, enrichment, and transportation pipeline. With connectors to common infrastructure.
- Kibana - data visualization platform.

“Elastic” provides complementary products that add more capabilities to the stack such as “shield” for security, “watcher” for alerts and more, but they are not open source, nor free.

### Elasticsearch

Elasticsearch is a search engine that is based on apache Lucene. Basically it is a NoSQL database that is adjusted for full text searches (like google search for instance). Elasticsearch is distributed, scalable, and supports high availability. It is very easy to setup and configure. It is very popular, and has a very good ecosystem.
Elasticsearch is also schemaless and stores the data in JSON documents (like MongoDB). It indexes all fields, it has many capabilities such as performing complicated text queries, highlight text, suggestions, geolocation and more.
It has a restful API and has many clients in many languages.
In addition to its full text search capabilities, Elasticsearch also supports time series data, and that’s make it is also a very good candidate for monitoring. Not only that we can perform monitoring, we can also perform full text search on logs. It adds another dimension without the need of another database.

### Logstash
Logstash is a data collector, basically what it does is:
Taking data from a source.
Filter the data and enrich it
Send it to the targets.
All those actions are configured in a file. 
Logstash has many inputs plugins that can get the data from many difference sources such as files, HTTP, log4j, syslog and many more. All you have to do is to configure the source and Logstash will take the data from there.
Logstash also has many plugins for filtering and manipulate the data, such as aggregation, parsing, conversion, and even a plugin that let you program with ruby.
Same goes for output, you can transfer the data to many outputs, the primary is of course Elasticsearch, but also to a file, redis, Kafka and even to InfluxDB and many more. If something is missing, there is guide for writing your own plugins.

### Data visualisation
 Kibana is more sophisticated than Chronograf. It has much more capabilities such as diversity of diagrams, geo location on map and more.

I would like to mention also [Grafana](http://grafana.org/) which is an excellent visualization tool, that can be used with both Elasticsearch and InfluxDB.

## Which one is better?
Both solutions are excellent, they are scalable, support high availability, easy to setup, configure  and maintain. And both of them are open source and free.
In a [performance test](https://www.influxdata.com/influxdb-markedly-elasticsearch-in-time-series-data-metrics-benchmark/) that InfluxData (the company that developed InfluxDB) performed, InfluxDB got much better results. But Elasticsearch can handle a huge load so for a regular system it can be enough.
Elasticsearch has an advantage over InfluxDB because you can use its full text search capability. This would be very useful if you want to save your logs (messages) and work with them. In order to get that capability with time series database like InfluxDB we will have to add another database that will support this service. It will make the system much more complicated. We will have to maintain two databases for monitoring, synchronize them in case in failure, probably add a message queue such as RabbitMQ, and all of this it with a high cost of time and money.
I think that the selection of the tool depends on the requirements. If the monitoring system should only monitor numbers through time, I would have picked InfluxDB, because it is more suited for the job. If you need also to save the logs or textual data, then pick Elasticsearch, in order to simplify the job.

## References
- https://en.wikipedia.org/wiki/Time_series_database
- https://www.influxdata.com/
- https://www.elastic.co/



