---
title: Writing Hudi Datasets
keywords: hudi, incremental, batch, stream, processing, Hive, ETL, Spark SQL
sidebar: mydoc_sidebar
permalink: writing_data.html
toc: false
summary: In this page, we will discuss some available tools for incrementally ingesting & storing data.
---

In this section, we will cover ways to ingest new changes from external sources or even other Hudi datasets using the [DeltaStreamer](#deltastreamer) tool, as well as 
speeding up large Spark jobs via upserts using the [Hudi datasource](#datasource-writer). Such datasets can then be [queried](querying_data.html) using various query engines.

## DeltaStreamer

The `HoodieDeltaStreamer` utility (part of hoodie-utilities) provides the way to ingest from different sources such as DFS or Kafka, with the following capabilities.

 - Exactly once ingestion of new events from Kafka, [incremental imports](https://sqoop.apache.org/docs/1.4.2/SqoopUserGuide.html#_incremental_imports) from Sqoop or output of `HiveIncrementalPuller` or files under a DFS folder
 - Support json, avro or a custom record types for the incoming data
 - Manage checkpoints, rollback & recovery 
 - Leverage Avro schemas from DFS or Confluent [schema registry](https://github.com/confluentinc/schema-registry).
 - Support for plugging in transformations

Command line options describe capabilities in more detail

```
[hoodie]$ spark-submit --class com.uber.hoodie.utilities.deltastreamer.HoodieDeltaStreamer `ls hoodie-utilities/target/hoodie-utilities-*-SNAPSHOT.jar` --help
Usage: <main class> [options]
  Options:
    --commit-on-errors
        Commit even when some records failed to be written
      Default: false
    --enable-hive-sync
          Enable syncing to hive
       Default: false
    --filter-dupes
          Should duplicate records from source be dropped/filtered outbefore 
          insert/bulk-insert 
      Default: false
    --help, -h
    --hoodie-conf
          Any configuration that can be set in the properties file (using the CLI 
          parameter "--propsFilePath") can also be passed command line using this 
          parameter 
          Default: []
    --key-generator-class
      Subclass of com.uber.hoodie.KeyGenerator to generate a HoodieKey from
      the given avro record. Built in: SimpleKeyGenerator (uses provided field
      names as recordkey & partitionpath. Nested fields specified via dot
      notation, e.g: a.b.c)
      Default: com.uber.hoodie.SimpleKeyGenerator
    --op
      Takes one of these values : UPSERT (default), INSERT (use when input is
      purely new data/inserts to gain speed)
      Default: UPSERT
      Possible Values: [UPSERT, INSERT, BULK_INSERT]
    --payload-class
      subclass of HoodieRecordPayload, that works off a GenericRecord.
      Implement your own, if you want to do something other than overwriting
      existing value
      Default: com.uber.hoodie.OverwriteWithLatestAvroPayload
    --props
      path to properties file on localfs or dfs, with configurations for
      Hudi client, schema provider, key generator and data source. For
      Hudi client props, sane defaults are used, but recommend use to
      provide basic things like metrics endpoints, hive configs etc. For
      sources, referto individual classes, for supported properties.
      Default: file:///Users/vinoth/bin/hoodie/src/test/resources/delta-streamer-config/dfs-source.properties
    --schemaprovider-class
      subclass of com.uber.hoodie.utilities.schema.SchemaProvider to attach
      schemas to input & target table data, built in options:
      FilebasedSchemaProvider
      Default: com.uber.hoodie.utilities.schema.FilebasedSchemaProvider
    --source-class
      Subclass of com.uber.hoodie.utilities.sources to read data. Built-in
      options: com.uber.hoodie.utilities.sources.{JsonDFSSource (default),
      AvroDFSSource, JsonKafkaSource, AvroKafkaSource, HiveIncrPullSource}
      Default: com.uber.hoodie.utilities.sources.JsonDFSSource
    --source-limit
      Maximum amount of data to read from source. Default: No limit For e.g:
      DFSSource => max bytes to read, KafkaSource => max events to read
      Default: 9223372036854775807
    --source-ordering-field
      Field within source record to decide how to break ties between records
      with same key in input data. Default: 'ts' holding unix timestamp of
      record
      Default: ts
    --spark-master
      spark master to use.
      Default: local[2]
  * --target-base-path
      base path for the target Hudi dataset. (Will be created if did not
      exist first time around. If exists, expected to be a Hudi dataset)
  * --target-table
      name of the target table in Hive
    --transformer-class
      subclass of com.uber.hoodie.utilities.transform.Transformer. UDF to
      transform raw source dataset to a target dataset (conforming to target
      schema) before writing. Default : Not set. E:g -
      com.uber.hoodie.utilities.transform.SqlQueryBasedTransformer (which
      allows a SQL query template to be passed as a transformation function)
```

The tool takes a hierarchically composed property file and has pluggable interfaces for extracting data, key generation and providing schema. Sample configs for ingesting from kafka and dfs are
provided under `hoodie-utilities/src/test/resources/delta-streamer-config`.

For e.g: once you have Confluent Kafka, Schema registry up & running, produce some test data using ([impressions.avro](https://docs.confluent.io/current/ksql/docs/tutorials/generate-custom-test-data.html) provided by schema-registry repo)

```
[confluent-5.0.0]$ bin/ksql-datagen schema=../impressions.avro format=avro topic=impressions key=impressionid
```

and then ingest it as follows.

```
[hoodie]$ spark-submit --class com.uber.hoodie.utilities.deltastreamer.HoodieDeltaStreamer `ls hoodie-utilities/target/hoodie-utilities-*-SNAPSHOT.jar` \
  --props file://${PWD}/hoodie-utilities/src/test/resources/delta-streamer-config/kafka-source.properties \
  --schemaprovider-class com.uber.hoodie.utilities.schema.SchemaRegistryProvider \
  --source-class com.uber.hoodie.utilities.sources.AvroKafkaSource \
  --source-ordering-field impresssiontime \
  --target-base-path file:///tmp/hoodie-deltastreamer-op --target-table uber.impressions \
  --op BULK_INSERT
```

In some cases, you may want to migrate your existing dataset into Hudi beforehand. Please refer to [migration guide](migration_guide.html). 

## Datasource Writer

The `hoodie-spark` module offers the DataSource API to write (and also read) any data frame into a Hudi dataset.
Following is how we can upsert a dataframe, while specifying the field names that need to be used
for `recordKey => _row_key`, `partitionPath => partition` and `precombineKey => timestamp`


```
inputDF.write()
       .format("com.uber.hoodie")
       .options(clientOpts) // any of the Hudi client opts can be passed in as well
       .option(DataSourceWriteOptions.RECORDKEY_FIELD_OPT_KEY(), "_row_key")
       .option(DataSourceWriteOptions.PARTITIONPATH_FIELD_OPT_KEY(), "partition")
       .option(DataSourceWriteOptions.PRECOMBINE_FIELD_OPT_KEY(), "timestamp")
       .option(HoodieWriteConfig.TABLE_NAME, tableName)
       .mode(SaveMode.Append)
       .save(basePath);
```

## Syncing to Hive

Both tools above support syncing of the dataset's latest schema to Hive metastore, such that queries can pick up new columns and partitions.
In case, its preferable to run this from commandline or in an independent jvm, Hudi provides a `HiveSyncTool`, which can be invoked as below, 
once you have built the hoodie-hive module.

```
cd hoodie-hive
./run_sync_tool.sh
 [hoodie-hive]$ ./run_sync_tool.sh --help
Usage: <main class> [options]
  Options:
  * --base-path
       Basepath of Hudi dataset to sync
  * --database
       name of the target database in Hive
    --help, -h
       Default: false
  * --jdbc-url
       Hive jdbc connect url
  * --pass
       Hive password
  * --table
       name of the target table in Hive
  * --user
       Hive username
```

## Storage Management

Hudi also performs several key storage management functions on the data stored in a Hudi dataset. A key aspect of storing data on DFS is managing file sizes and counts
and reclaiming storage space. For e.g HDFS is infamous for its handling of small files, which exerts memory/RPC pressure on the Name Node and can potentially destabilize
the entire cluster. In general, query engines provide much better performance on adequately sized columnar files, since they can effectively amortize cost of obtaining 
column statistics etc. Even on some cloud data stores, there is often cost to listing directories with large number of small files.

Here are some ways to efficiently manage the storage of your Hudi datasets.

 - The [small file handling feature](configurations.html#compactionSmallFileSize) in Hudi, profiles incoming workload 
   and distributes inserts to existing file groups instead of creating new file groups, which can lead to small files. 
 - Cleaner can be [configured](configurations.html#retainCommits) to clean up older file slices, more or less aggressively depending on maximum time for queries to run & lookback needed for incremental pull
 - User can also tune the size of the [base/parquet file](configurations.html#limitFileSize), [log files](configurations.html#logFileMaxSize) & expected [compression ratio](configurations.html#parquetCompressionRatio), 
   such that sufficient number of inserts are grouped into the same file group, resulting in well sized base files ultimately.
 - Intelligently tuning the [bulk insert parallelism](configurations.html#withBulkInsertParallelism), can again in nicely sized initial file groups. It is in fact critical to get this right, since the file groups
   once created cannot be deleted, but simply expanded as explained before.
 - For workloads with heavy updates, the [merge-on-read storage](concepts.html#merge-on-read-storage) provides a nice mechanism for ingesting quickly into smaller files and then later merging them into larger base files via compaction.