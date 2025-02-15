## Introduction

Driving is one of the most common yet dangerous tasks that people
perform every day. According to the NHTSA, there is an average of 6
million accidents in the U.S. per year, resulting in over 2.35 million
injuries and 37,000 deaths. [1] Additionally, road crashes cost the U.S.
$230.6 billion per year. Various factors can contribute to accidents
such as distracted driving, speeding, poor weather conditions, and
alcohol involvement. However, using these factors and additional
statistics, we can better predict the cause of accidents and put laws
and procedures in place to help minimize the number of accidents. For
this assignment, we used Apache Spark to analyze the motor vehicle
collisions in New York City (NYC). Our goal was to use Spark to gain
additional insights into the causes of accidents in the Big Apple, which
can be used to help prevent certain accidents from occurring.

## Datasets

The NYC Motor Vehicle Collisions - Crashes [2] dataset was the primary
dataset used for our analysis. We also used NYC Motor Vehicle Crashes -
Vehicle [3] and the 2015 NYC Tree Census [4] as our secondary datasets.
All of the datasets were obtained from NYC OpenData in CSV format and
contain the most up to date motor vehicle collision information
available to the public.

### NYC Motor Vehicle Collisions - Crashes:

This dataset contains information about the motor vehicle collisions in
NYC from July 2012 to February 2020. The data was extracted from police
reports (form MV104-AN) that were filed at the time of the crash. A form
MV104-AN is only filed in the case where an individual is injured or
fatally injured, or when the damage caused by the accident is $1,000 or
greater. This dataset has 29 columns, 1.65 Million rows, and a size of
369 MB. Each row includes details about a specific motor vehicle
collision.

### NYC Motor Vehicle Crashes - Vehicle:

This dataset contains information about each vehicle that was involved
in a crash in NYC from September 2012 to May 2020 and a police report
MV104-AN was filed. This dataset has 3.35M rows, 25 columns, and a size
of 566.3 MB. Each row represents the vehicle information for a specific
crash, which can also be tied back to the NYC Motor Vehicle Collisions -
Crashes dataset. Multiple vehicles can be involved in a single crash.

### 2015 NYC Tree Census:

This dataset contains detailed information on the trees living
throughout NYC collected by the NYC Parks and Recreation Board in 2015.
This dataset has 684K rows, 45 columns, and has a size of 220.4 MB. Each
row represents the information for a tree living in NYC.

## Chosen Spark API for Answering Analytical Questions

I chose to use the Spark DataFrames API for my analysis. The Spark
DataFrames API is similar to relational tables in SQL, however, it also
provides a programmatic API allowing for more flexibility in query
expressiveness, query testing, and is extensible. Additionally, the
datasets that were chosen for our analysis are in a format that can be
easily worked with using DataFrames. For example, our data is in a
tabular format with columns and rows, which is how data is represented
in DataFrames. Additionally, DataFrames inherits all of the properties
of RDDs such as read-only, fault tolerance, caching, and lazy execution
but with the additional data storage optimizations, code generation,
query planner, and abstraction.

## Loading Datasets

In the beforeAll() function, all three CSV files are read in as as
DataFrames then compressed to Parquet format and persisted in external
storage. A check is made to determine if the Parquet files exist on disk
before running a test. If the files already exist, then they will be
reused for all subsequent tests. Otherwise, they will be regenerated.
Through the use of Parquet, our data will be stored in a compressed
columnar format, which will allow for faster read and write performance
as compared to reading from CSV. The file size of the datasets when
converted to Parquet, are significantly smaller as compared to the
original CSV files. For example, the CSV file for the NYC Motor Vehicle
\-Crashes was 369 MB and was reduced to 75.7 MB when converted to
Parquet. Additionally, the output Parquet files for the NYC Motor
Vehicle - Vehicles dataset was partitioned by ZIP_CODE. By doing this,
the data is physically laid out on the filesystem in an order that will
be the most efficient for performing our queries.

Some data preparation was needed before converting to Parquet, which
includes the following.
- For all DataFrames, the whitespace between columns names needed to be
  replaced with underscores to avoid the invalid character errors when
  converting to Parquet. E.g., from CRASH TIME to CRASH_TIME.
- The data types of several columns in the NYC Motor Vehicle - Crashes
  dataset needed to be cast from a string to an integer type because
  they are used for numerical calculations in our query.

The below Directed acyclic graph (DAG) shows the operations performed
when reading a parquet file. This operation is performed each time a
test runs. Spark performs a parallelize operation followed by a
mapPartitions operation.

![Parquet1](data/Images/Parquet1.png)
![Parquet2](data/Images/Parquet2.png)

## Analytic Questions

### 1. What are the top five most frequent contributing factors for accidents in NYC?

For this test, we first filtered out rows with an unspecified
contributing factor. Next, we perform a groupBy
CONTRIBUTING_FACTOR_VEHICLE_1, then count the number of occurrences of
each contributing factor. Finally, we order by count and get the top 5
contributing factors.

#### Spark Internal Analysis

There are four stages that occur for this problem. In stage 0, we get
the listing leaf files and directory for 234 paths, which are the
different paths that were created when we partitioned the NYC Motor
Vehicle Collisions \- Crashes DataFrame by "ZIP_CODE". Each path
represents a different ZIP_CODE. Next, in stage 1, we read the Parquet
file from disk (as seen in the previous section). The RDDs created in
stage 1 are a parrallelCollectionRDD followed by a mapPartitionsRDD.
Next, in stage 2 a FileScanRDD is created, followed by a
MapPartitionRDD, then another MapParitionsRDD. In stage 3 a
shuffledRowRDD is created because we are performing a groupBy function
which is a wide transformation, and thus data is shuffled across the
cluster of nodes. Next, a MapPartitionsRDD is created, followed by
another mapPartitionsRDD in the map step. The map function uses a narrow
transformation, so the computations do not need to be shuffled across
the cluster.

![Problem 1 cropped](data/Images/Problem%201%20cropped.png)
![Problem 1 Stage 2 cropped](data/Images/Problem%201%20Stage%202%20cropped.png)
![Problem 1 Stage 3 cropped](data/Images/Problem%201%20Stage%203%20cropped.png)

### 2. What percentage of accidents had alcohol as a contributing factor?

For this test, we first got the count of total accidents and stored in a
val numTotalAccidents. Next, we filtered out accidents where alcohol
involvement was a contributing factor for any of the vehicles involved
in the accident and stored the result in numAlcoholRelatedAccidents.
Finally, we performed the following calculation to get the percentage.

```scala
(numAlcoholRelatedAccidents * 100) / numTotalAccidents.toDouble 
```

#### Spark Internal Analysis

The data in stage 2 was split into 27 partitions, each executing a task.
However, the data in stage 3 is only needed to execute a single task on
a single partition.

![Problem 2](data/Images/Problem%202.png)
![Problem 2 stage 2](data/Images/Problem%202%20stage%202.png)
![Problem 2 Stage 3](data/Images/Problem%202%20Stage%203.png)

### 3. What time of day sees the most cyclist injures or deaths caused by a motor vehicle collision?

For this test, we first filter out accidents where there was at least
one cyclist injury or fatality. Next, we perform a groupBy("CRASH_TIME")
and counting the number of crashes for the various times throughout a 24
hour period. Finally, we order by count in descending order and get the
top 3 times.

#### Spark Internal Analysis

As we can see from stage 2, the tasks each took various amounts of time,
some longer and some shorter. Additionally, towards the end of each
task, a shuffle write was performed because the operation performed
required a wide transformation. In this case, each task spends some time
writing the results across the cluster. In total stage 2 took 1.1 minute
to complete all of its 27 tasks. On the other hand, stage 3 executed
over 200 tasks with a total time across all tasks of only 13 seconds. By
this, we can tell that shuffling is costly when it comes to execution
time, especially if we are processing large amounts of data.

![Problem 3](data/Images/Problem%203.png)
![Problem 3 Stage 2](data/Images/Problem%203%20Stage%202.png)
![Problem 3 Stage 3](data/Images/Problem%203%20Stage%203.png)
![Problem 3 Stage 3_1](data/Images/Problem%203%20Stage%203_1.png)

### 4. Which zip code had the largest number of nonfatal and fatal accidents?

Finding the zip code with the most significant number of nonfatal and
fatal accidents required several steps. First, remove rows with null zip
codes. Next, create a new column "TOTAL_INJURED_OR_KILLED", which is the
sum of all nonfatal and fatal injuries for each accident. Next, get the
total number of nonfatal and fatal injuries per zip code by first
performing a group by zip code. Then use the agg and sum functions to
sum up the values in the TOTAL_INJURED_OR_KILLED column per zip code.
Afterward, store the computed output per zip code in a column aliased as
TOTAL_INJURIES_AND_FATALITIES. Finally, order the results by
TOTAL_INJURIES_AND_FATALITIES in descending order and get the top 3.

#### Spark Internal Analysis

Because we are calling the filter function right after loading the
dataset, Spark performed a predicate pushdown, where it pushed the
filter function down to the data source and performed the filter query
before returning the result to the driver. A predicate pushdown improves
query performance because filtering is done before loading the dataset
in the driver, so less data is returned.


![Problem 4](data/Images/Problem%204.png)
![Problem 4 stage 2](data/Images/Problem%204%20stage%202.png)
![Problem 4 stage 3](data/Images/Problem%204%20stage%203.png)
![Problem 4 stage 3_1](data/Images/Problem%204%20stage%203_1.png)

### 5. Which vehicle make, model, and year was involved in the most accidents?

To determine which vehicle make, model, and year was involved in the
most accidents, we first remove any rows where the vehicle make, model,
and year is null. Next, we did a groupBy("VEHICLE_MAKE",
"VEHICLE_MODEL", "VEHICLE_YEAR"), then count and sort by descending
order and get the first value, which is the most accidents. Likewise, to
get the least accidents, we did the same previous steps but sort by
ascending order and got the first value.

#### Spark Internal Analysis

Similar to test 4, test 5 is also performing a filter as the first
function in the test, so Spark performed a predicate pushdown as well.
Additionally, in stage 3, we reused the cached RDD that was created from
performing a previous transformation. Reusing the cached RDD allowed for
faster access time in future actions that needed to use the same data.

![Problem 5](data/Images/Problem%205.png)
![Problem 5 Stage 3](data/Images/Problem%205%20Stage%203.png)
![Problem 5 stage 4](data/Images/Problem%205%20stage%204.png)

### 6. How do the number of collisions in an area of NYC correlate to the number of trees in the area?

We used the NYC motor vehicle collisions - crashes and 2015 NYC tree
census datasets for this test. We first rename the postcode column to
ZIP_CODE on the treeCensus DataFrame. Next, we perform a groupBy on the
treeCensus DataFrame to get the number of trees per zip code. We also
performed a groupBy on the collisions DataFrame to get the number of
accidents per zip code. Finally, we did an inner equi-join of the
collisions DataFrame with the treeCensus DataFrame using the "ZIP_CODE"
collumn. Then finally, ordered by "TOTAL_CRASHES" in descending order.

#### Spark Internal Analysis

The previous tests only needed to load a single DataFrame and had two to
three parquet jobs per test. However, this test uses two DataFrames and
thus required a third parquet job to read the additional data. In stage
5, the renaming of the postcode column is performed, followed by the
groupBy on the treeCensus DataFrame which reused the cached treeCensus
data. This test also used a join function, which performs a wide
transformation, and therefore shuffling was needed.
![Problem 6 jobs](data/Images/Problem%206%20jobs.png)
![Problem 6 stages](data/Images/Problem%206%20stages.png)

![Problem 6 stage 5](data/Images/Problem%206%20stage%205.png)
![Problem 6 stage 6](data/Images/Problem%206%20stage%206.png)

The code ran on a single executor with 4 RDD Blocks, 4 cores CPU, and
1.1 GB RAM.

![Problem 6 executors](data/Images/Problem%206%20executors.png)

Since we are caching the Parquet files for both the collisions and
treeCensus DataFrames, an RDD is created for each dataset and persisted
to memory for reuse during subsequent actions in the test. The
transformed RDD for the treeCensus data is cached into 4 partitions and
uses 82.3 MB in RAM storage. Additionally, the transformed RDD for the
collisions data is cached into 27 partitions and takes up 154.8 MB in
RAM storage. Because Sparks transformations are lazy, the
transformations will not execute until an action is performed on it.
However, by adding caching, we are ensuring that when actions are
executed on transformations that use the same data, that we are reusing
the previously processed data vs. having to reload the data.

![Problem 6 Storage](data/Images/Problem%206%20Storage.png)

## Project Overview

- Language: [Scala](https://www.scala-lang.org/)
- Framework: [Apache Spark](https://spark.apache.org/)
- Build tool: [SBT](https://www.scala-sbt.org/)
- Testing Framework: [Scalatest](http://www.scalatest.org/)

## Running Tests

### From Intellij

Right click on `ExampleDriverTest` and choose `Run 'ExampleDriverTest'`

### From the command line

On Unix systems, test can be run:

```shell script
$ ./sbt test
```

or on Windows systems:

```shell script
C:\> ./sbt.bat test
```

## Configuring Logging

Spark uses log4j 1.2 for logging. Logging levels can be configured in
the file `src/test/resources/log4j.properties`

Spark logging can be verbose, for example, it will tell you when each
task starts and finishes as well as resource cleanup messages. This
isn't always useful or desired during regular development. To reduce the
verbosity of logs, change the line `log4j.logger.org.apache.spark=INFO`
to `log4j.logger.org.apache.spark=WARN`

## Documentation

* RDD: https://spark.apache.org/docs/latest/rdd-programming-guide.html
* Batch Structured APIs:
  https://spark.apache.org/docs/latest/sql-programming-guide.html

## References

[1] National Highway Traffic Safety Administration. “NCSA Publications
&amp; Data Requests.” Early Estimate of Motor Vehicle Traffic Fatalities
for the First Quarter of 2019, 2019,
https://crashstats.nhtsa.dot.gov/Api/Public/ViewPublication/812783.

[2] (NYPD), Police Department. “Motor Vehicle Collisions - Crashes: NYC
Open Data.” Motor Vehicle Collisions - Crashes | NYC Open Data, 8 May
2020,
data.cityofnewyork.us/Public-Safety/Motor-Vehicle-Collisions-Crashes/h9gi-nx95.

[3] (NYPD), Police Department. “Motor Vehicle Collisions - Vehicles: NYC
Open Data.” Motor Vehicle Collisions - Vehicles | NYC Open Data, 8 May
2020,
data.cityofnewyork.us/Public-Safety/Motor-Vehicle-Collisions-Vehicles/bm4k-52h4.

[4] Department of Parks and Recreation. “2015 Street Tree Census - Tree
Data: NYC Open Data.” 2015 Street Tree Census - Tree Data | NYC Open
Data, 4 Oct. 2017,
data.cityofnewyork.us/Environment/2015-Street-Tree-Census-Tree-Data/uvpi-gqnh.

