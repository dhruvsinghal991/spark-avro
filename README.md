# Spark SQL Avro Library

A library for reading and writing Avro data from [Spark SQL](http://spark.apache.org/docs/latest/sql-programming-guide.html).

[![Build Status](https://travis-ci.org/databricks/spark-avro.svg?branch=master)](https://travis-ci.org/databricks/spark-avro)
[![codecov.io](http://codecov.io/github/databricks/spark-avro/coverage.svg?branch=master)](http://codecov.io/github/databricks/spark-avro?branch=master)

## Changes

The databricks avro library does a UNION RDD on all the input files provided to the build scan, infact it creates an RDD for every file input to the HadoopFsRelation. This requires a lot of driver memory to broadcast large amount of RDDs in case of highly partitioned AVRO files although this is resolved in the case of spark version > 2.0.0. The class `org.apache.avro.mapred.AvroInputFormat` can by itself handle a single file, directory or glob. 



## Requirements

This documentation is for Spark 1.4+.

This library has different versions for Spark 1.2, 1.3, and 1.4+:

| Spark Version | `spark-avro` version |
| ------------- |----------------------|
| `1.2`         | `0.2.0`              |
| `1.3`         | `1.0.0`              |
| `1.4+`        | `2.0.0`              |

## Linking

You can link against this library (for Spark 1.4+) in your program at the following coordinates:

Using SBT:

```
libraryDependenicies += "com.databricks" %% "spark-avro" % "2.0.0"
```

Using Maven:

```xml
<dependency>
    <groupId>com.databricks<groupId>
    <artifactId>spark-avro_2.10</artifactId>
    <version>2.0.0</version>
</dependency>
```

The `spark-avro` library can also be added to Spark jobs launched through `spark-shell` or `spark-submit` by using the `--packages` command line option.
For example, to include it when starting the spark shell:

```
$ bin/spark-shell --packages com.databricks:spark-avro_2.10:2.0.0
```

Unlike using `--jars`, using `--packages` ensures that this library and its dependencies will be added to the classpath.
The `--packages` argument can also be used with `bin/spark-submit`.

This library is cross-published for Scala 2.11, so 2.11 users should replace 2.10 with 2.11 in the commands listed above.

## Features

`spark-avro` supports reading and writing of Avro data from Spark SQL.

- **Automatic schema conversion:** `spark-avro` supports most conversions between Spark SQL and Avro records, making Avro a first-class citizen in Spark.
- **Partitioning:** This library allows developers to easily read and write partitioned data
witout any extra configuration. Just pass the columns you want to partition on, just like you would for Parquet.
- **Compression:**  You can specify the type of compression to use when writing Avro out to
disk. The supported types are `uncompressed`, `snappy`, and `deflate`. You can also specify the deflate level.
- **Specifying record names:** You can specify the record name and namespace to use by passing a map of parameters with `recordName` and `recordNamespace`.

## Supported types for Avro -> Spark SQL conversion

`spark-avro` supports reading all Avro types, with the exception of complex `union` types. It uses the following mapping from Avro types to Spark SQL types:

| Avro type | Spark SQL type |
| --------- |----------------|
| boolean   | BooleanType    |
| int       | IntegerType    |
| long      | LongType       |
| float     | FloatType      |
| double    | DoubleType     |
| bytes     | BinaryType     |
| string    | StringType     |
| record    | StructType     |
| enum      | StringType     |
| array     | ArrayType      |
| map       | MapType        |
| fixed     | BinaryType     |

In addition to the types listed above, `spark-avro` supports reading of three types of `union` types:

1. `union(int, long)`
2. `union(float, double)`
3. `union(something, null)`, where `something` is one of the supported Avro types listed above or is one of the supported `union` types.

At the moment, `spark-avro` ignores docs, aliases and other properties present in the Avro file.

## Supported types for Spark SQL -> Avro conversion

`spark-avro` supports writing of all Spark SQL types into Avro. For most types, the mapping from Spark types to Avro types is straightforward (e.g. IntegerType gets converted to int); however, there are a few special cases which are listed below:

| Spark SQL type | Avro type |
| ---------------|-----------|
| ByteType       | int       |
| ShortType      | int       |
| DecimalType    | string    |
| BinaryType     | bytes     |
| TimestampType  | long      |
| StructType     | record    |

## Examples

The recommended way to read or write Avro data from Spark SQL is by using Spark's DataFrame APIs, which are available in Scala, Java, Python, and R.

These examples use an Avro file available for download
[here](https://github.com/databricks/spark-avro/raw/master/src/test/resources/episodes.avro):

### Scala API

```scala
// import needed for the .avro method to be added
import com.databricks.spark.avro._
		
val sqlContext = new SQLContext(sc)

// The Avro records get converted to Spark types, filtered, and
// then written back out as Avro records
val df = sqlContext.read.avro("src/test/resources/episodes.avro")
df.filter("doctor > 5").write.avro("/tmp/output")
```

Alternativly you can specify the format to use instead:

```scala
val sqlContext = new SQLContext(sc)
val df = sqlContext.read
	.format("com.databricks.spark.avro")
	.load("src/test/resources/episodes.avro")
	
df.filter("doctor > 5").write
	.format("com.databricks.spark.avro")
	.save("/tmp/output")
```

You can also specify Avro compression options:

```scala
import com.databricks.spark.avro._
val sqlContext = new SQLContext(sc)

// configuration to use deflate compression
sqlContext.setConf("spark.sql.avro.compression.codec", "deflate")
sqlContext.setConf("spark.sql.avro.deflate.level", "5")

val df = sqlContext.read.avro("src/test/resources/episodes.avro")

// writes out compressed Avro records
df.write.avro("/tmp/output")
```

You can write partitioned Avro records like this:

```scala
import com.databricks.spark.avro._

val sqlContext = new SQLContext(sc)

import sqlContext.implicits._

val df = Seq((2012, 8, "Batman", 9.8),
	(2012, 8, "Hero", 8.7),
	(2012, 7, "Robot", 5.5),
	(2011, 7, "Git", 2.0))
	.toDF("year", "month", "title", "rating")

df.write.partitionBy("year", "month").avro("/tmp/output")
```

You can specify the record name and namespace like this:

```scala
import com.databricks.spark.avro._

val sqlContext = new SQLContext(sc)
val df = sqlContext.read.avro("src/test/resources/episodes.avro")

val name = "AvroTest"
val namespace = "com.databricks.spark.avro"
val parameters = Map("recordName" -> name, "recordNamespace" -> namespace)

df.write.options(parameters).avro("/tmp/output")
```

### Java API

```java
import org.apache.spark.sql.*;

SQLContext sqlContext = new SQLContext(sc);

// Creates a DataFrame from a specified file
DataFrame df = sqlContext.read().format("com.databricks.spark.avro")
    .load("src/test/resources/episodes.avro");

// Saves the subset of the Avro records read in
df.filter($"age > 5").write()
	.format("com.databricks.spark.avro")
	.save("/tmp/output");
```


### Python API

```python
# Creates a DataFrame from a specified directory
df = sqlContext.read.format("com.databricks.spark.avro").load("src/test/resources/episodes.avro")

#  Saves the subset of the Avro records read in
subset = df.where("age > 5")
subset.write.format("com.databricks.spark.avro").save("output")
```

### SQL API
Avro data can be queried in pure SQL by registering the data as a temporary table.

```sql
CREATE TEMPORARY TABLE episodes
USING com.databricks.spark.avro
OPTIONS (path "src/test/resources/episodes.avro")
```



## Building From Source
This library is built with [SBT](http://www.scala-sbt.org/0.13/docs/Command-Line-Reference.html),
which is automatically downloaded by the included shell script.  To build a JAR file simply run
`build/sbt package` from the project root.

## Testing
To run the tests, you should run `build/sbt test`. In case you are doing improvements that target
speed, you can generate a sample Avro file and check how long it takes to read that Avro file
using the following commands:

```
build/sbt "test:run-main com.databricks.spark.avro.AvroFileGenerator NUMBER_OF_RECORDS NUMBER_OF_FILES"
```

will create sample avro files in `target/avroForBenchmark/`. You can specify the number of records
for each file, as well as the overall number of files.

```
build/sbt "test:run-main com.databricks.spark.avro.AvroReadBenchmark"
```

runs `count()` on the data inside `target/avroForBenchmark/` and tells you how the operation took.

Similarly, you can do benchmarks on how long it takes to write DataFrame as Avro file with

```
build/sbt "test:run-main com.databricks.spark.avro.AvroWriteBenchmark NUMBER_OF_ROWS"
```

where `NUMBER_OF_ROWS` is an optional parameter that allows you to specify the number of rows in DataFrame that we will be writing.
