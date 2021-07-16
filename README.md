# DeltaLiveTables
Delta Live Tables concepts
In this section:
Pipelines
Datasets
Continuous and triggered pipelines
Complete tables in continuous pipelines
Maintenance tasks
Development and production modes
Pipelines
The main unit of execution in Delta Live Tables is a pipeline. A pipeline is a directed acyclic graph (DAG) linking data sources to target datasets. You define the contents of Delta Live Tables datasets using SQL queries or Python functions that return Spark SQL or Koalas DataFrames. A pipeline also has an associated configuration defining the settings required to run the pipeline. You can optionally specify data quality constraints when defining datasets.

In this section:
Queries
Expectations
Pipeline settings
Pipeline updates
Queries
Queries implement data transformations by defining a data source and a target dataset. Delta Live Tables queries can be implemented in Python or SQL.

Expectations
You use expectations to specify data quality controls on the contents of a dataset. Unlike a CHECK constraint in a traditional database which prevents adding any records that fail the constraint, expectations provide flexibility when processing data that fails data quality requirements. This flexibility allows you to process and store data that you expect to be messy and data that must meet strict quality requirements.

You can define expectations to retain records that fail validation, drop records that fail validation, or halt the pipeline when a record fails validation.

Pipeline settings
Pipeline settings are defined in JSON and include the parameters required to run the pipeline, including:

Libraries (in the form of notebooks) that contain the queries that describe the tables and views to create the target datasets in Delta Lake.
A cloud storage location where the tables and metadata required for processing will be stored. This location is either DBFS or another location you provide.
Other required dependencies, for example, PyPI dependencies.
Optional configuration for a Spark cluster where data processing will take place.
See Delta Live Tables settings for more details.

Pipeline updates
After you create the pipeline and are ready to run it, you start an update. An update:

Starts a cluster with the correct configuration.
Discovers all the tables and views defined, and checks for any analysis errors such as invalid column names, missing dependencies, syntax errors, and so on.
Creates or updates all of the tables and views with the most recent data available.
If the pipeline is triggered, the system stops processing after updating all tables in the pipeline once.

When a triggered update completes successfully, each table is guaranteed to be updated based on the data available when the update started.

For use cases that require low latency, you can configure a pipeline to update continuously.

See Continuous and triggered pipelines for more information about choosing an execution mode for your pipeline.

Datasets
There are two types of datasets in a Delta Live Tables pipeline: views and tables.

Views are similar to a temporary view in SQL and are an alias for some computation. A view allows you to break a complicated query into smaller or easier-to-understand queries. Views also allow you to reuse a given transformation as a source for more than one table. Views are available from within a pipeline only and cannot be queried interactively.
Tables are similar to traditional materialized views. The Delta Live Tables runtime automatically creates tables in the Delta format and ensures those tables are updated with the latest result of the query that creates the table.
Tables can be incremental or complete. Incremental tables support updates based on continually arriving data without having to recompute the entire table. A complete table is entirely recomputed with each update.

You can publish your tables to make them available for discovery and querying by downstream consumers.

Temporary tables
To prevent publishing of intermediate tables that are not intended for external consumption, mark them as TEMPORARY:

SQL

Copy
CREATE TEMPORARY LIVE TABLE temp_table
AS SELECT...
Continuous and triggered pipelines
Delta Live Tables supports two different modes of execution:

Triggered pipelines update each table with whatever data is currently available and then stop the cluster running the pipeline. Delta Live Tables automatically analyzes the dependencies between your tables and starts by computing those that read from external sources. Tables within the pipeline are updated after their dependent data sources have been updated.
Continuous pipelines update tables continuously as input data changes. Once an update is started, it continues to run until manually stopped. Continuous pipelines require an always-running cluster but ensure that downstream consumers have the most up-to-date data.
Triggered pipelines can reduce resource consumption and expense since the cluster runs only long enough to execute the pipeline. However, new data won’t be processed until the pipeline is triggered. Continuous pipelines require an always-running cluster, which is more expensive but reduces processing latency.

The continuous flag in the pipeline settings controls the execution mode. Pipelines run in triggered execution mode by default. Set continuous to true if you require low latency updates of the tables in your pipeline.


Copy
{
  ...
  "continuous": “true”,
  ...
}
The execution mode is independent of the type of table being computed. Both complete and incremental tables can be updated in either execution mode.

If some tables in your pipeline have weaker latency requirements, you can configure their update frequency independently by setting the pipelines.trigger.interval setting:

Python

Copy
spark_conf={"pipelines.trigger.interval", "1 hour"}
This option does not turn off the cluster in between pipeline updates, but can free up resources for updating other tables in your pipeline.

Complete tables in continuous pipelines
Complete tables can also be present in a pipeline that runs continuously. To avoid unnecessary processing, pipelines automatically monitor dependent Delta tables and perform an update only when the contents of those dependent tables have changed.

The Delta Live Tables runtime is not able to detect changes in non-Delta data sources. The table is still updated regularly, but with a higher default trigger interval to prevent excessive recomputation from slowing down any incremental processing happening on the cluster.

Maintenance tasks
Delta Live Tables performs maintenance tasks on tables daily, including OPTIMIZE and VACUUM. These operations can improve query performance and reduce cost by removing old versions of tables.

You can control retention periods, table optimization, and Z-Order configurations using table properties.

Development and production modes
You can optimize pipeline execution by switching between development and production modes. When you run your pipeline in development mode, the Delta Live Tables system:

Reuses a cluster to avoid the overhead of restarts.
Disables pipeline retries so you can immediately detect and fix errors.
In production mode, the Delta Live Tables system:

Restarts the cluster for specific recoverable errors, including memory leaks and stale credentials.
Retries execution in the event of specific errors, for example, a failure to start a cluster.
Use the Delta Live Tables Environment Toggle Icon buttons in the Pipelines UI to switch between development and production modes. By default, pipelines run in production mode.

Publish tables
You can make the output data of your pipeline discoverable and available to query by publishing datasets to the Azure Databricks metastore. To publish datasets to the metastore, specify a target database in the Delta Live Tables settings:


Copy
{
  ...
  "target": "prod_customer_data"
  ...
}
When you configure the target option, the Delta Live Tables runtime publishes all the tables in the pipeline and their associated metadata. For this example, a table called sales will be available to query using the name prod_customer_data.sales.

You can use this feature with multiple environment configurations to publish to different databases based on the environment. For example, you can publish to a dev database for development and a prod database for production data.

When you create a target configuration, only tables and associated metadata are published. Views are not published to the metastore.

Implement pipelines
Notebooks
You implement Delta Live Tables pipelines in Azure Databricks notebooks. You can implement pipelines in a single notebook or in multiple notebooks. All queries in a single notebook must be implemented in either Python or SQL, but you can configure multiple-notebook pipelines with a mix of Python and SQL notebooks. Each notebook shares a storage location for output data and is able to reference datasets from other notebooks in the pipeline.

See Delta Live Tables quickstart to learn more about creating and running a pipeline. See Configure multiple notebooks in a pipeline for an example of configuring a multi-notebook pipeline.

Queries
You can implement Delta Live Tables queries in Python or SQL. See the Delta Live Tables language reference for more information about the supported languages.

Python
Apply the @view or @table decorator to a function to define a view or table in Python. You can use the function name or the name parameter to assign the table or view name. The following example defines two different datasets: a view called taxi_raw that takes a JSON file as the input source and a table called filtered_data that takes the taxi_raw view as input:

Python

Copy
@dlt.view
def taxi_raw():
  return spark.read.json("/databricks-datasets/nyctaxi/sample/json/")

# Use the function name as the table name
@dlt.table
def filtered_data():
  return dlt.read("taxi_raw").where(...)

# Use the name parameter as the table name
@dlt.table(
  name="filtered_data")
def create_filtered_data():
  return dlt.read("taxi_raw").where(...)
View and table functions must return a Spark DataFrame or Koalas DataFrame. A Koalas DataFrame returned by a function is converted to a Spark Dataset by the Delta Live Tables runtime.

In addition to reading from external data sources, you can use the read() function to access other datasets defined in your pipeline. The read() function ensures that the pipeline automatically captures the dependency between datasets. This dependency information is used to determine the execution order when performing an update and recording lineage information in the event log for a pipeline.

Both views and tables have the following optional properties:

comment: A human-readable description of this dataset.
spark_conf: A Python dictionary containing Spark configurations for the execution of this query only.
Data quality constraints enforced with expectations.
Tables also offer additional control of their materialization:

You can specify how tables are partitioned using partition_cols. Partitioning can be used to speed up queries.
You can set table properties when you define a view or table. See Table properties for more details.
By default, table data is stored in the pipeline storage location. You can set an alternate storage location using the path setting.
The Python API is defined in the dlt module. See Python in the Delta Live Tables language reference for more information about the Python API and dataset properties.

You can use external Python dependencies in a Delta Live Tables Python notebook. For more information, see Notebook-scoped Python libraries.

SQL
Use the CREATE LIVE VIEW or CREATE LIVE TABLE syntax to create a view or table with SQL. You can create a dataset by reading from an external data source or from datasets defined in a pipeline. You prepend the LIVE keyword to a dataset name to read from an internal dataset. The following example defines two different datasets: a view called taxi_raw that takes a JSON file as the input source and a table called filtered_data that takes the taxi_raw view as input:

SQL

Copy
CREATE LIVE TABLE taxi_raw
AS SELECT * FROM json.`/databricks-datasets/nyctaxi/sample/json/`

CREATE LIVE TABLE filtered_data
AS SELECT
  ...
FROM LIVE.taxi_raw
Delta Live Tables automatically captures the dependencies between datasets defined in your pipeline and uses this dependency information to determine the execution order when performing an update and to record lineage information in the event log for a pipeline.

Both views and tables have the following optional properties:

COMMENT: A human-readable description of this dataset.
Data quality constraints enforced with expectations.
Tables also offer additional control of their materialization:

You can specify how tables are partitioned using PARTITIONED BY. Partitioning can be used to speed up queries.
You can set table properties using TBLPROPERTIES. See Table properties for more detail.
By default, table data is stored in the pipeline storage location. You can set an alternate storage location using the LOCATION setting.
See SQL in the Delta Live Tables language reference for more information about table and view properties.

Define data quality constraints
You use expectations to define data quality constraints on the contents of a dataset. An expectation consists of a description, an invariant, and an action to take when a record fails the invariant. You apply expectations to queries using Python decorators or SQL constraint clauses.

Use the expect, expect or drop, and expect or fail expectations with Python or SQL queries to define a single data quality constraint.

You can define expectations with one or more data quality constraints in Python pipelines using the @expect_all, @expect_all_or_drop, and @expect_all_or_fail decorators. These decorators accept a Python dictionary as an argument, where the key is the expectation name and the value is the expectation constraint.

Retain invalid records
Use the expect operator when you want to keep records that violate the expectation. Records that violate the expectation are added to the target dataset along with valid records. The number of records that violate the expectation can be viewed in data quality metrics for the target dataset:

Python
Python

Copy
@dlt.expect("valid timestamp", "col(“timestamp”) > '2012-01-01'")
SQL
SQL

Copy
CONSTRAINT valid_timestamp EXPECT (timestamp > '2012-01-01')
Drop invalid records
Use the expect or drop operator to prevent the processing of invalid records. Records that violate the expectation are dropped from the target dataset. The number of records that violate the expectation can be viewed in data quality metrics for the target dataset:

Python
Python

Copy
@dlt.expect_or_drop("valid_current_page", "current_page_id IS NOT NULL AND current_page_title IS NOT NULL")
SQL
SQL

Copy
CONSTRAINT valid_current_page EXPECT (current_page_id IS NOT NULL and current_page_title IS NOT NULL) ON VIOLATION DROP ROW
Fail on invalid records
When invalid records are unacceptable, use the expect or fail operator to halt execution immediately when a record fails validation. If the operation is a table update, the system atomically rolls back the transaction:

Python
Python

Copy
@dlt.expect_or_fail("valid_count", "count > 0")
SQL
SQL

Copy
CONSTRAINT valid_count EXPECT (count > 0) ON VIOLATION FAIL UPDATE
When a pipeline fails because of an expectation violation, you must fix the pipeline code to handle the invalid data correctly before re-running the pipeline.

Fail expectations modify the Spark query plan of your transformations to track information required to detect and report on violations. For many queries, you can use this information to identify which input record resulted in the violation. The following is an example exception:

Console

Copy
Expectation Violated:
{
  "flowName": "a-b",
  "verboseInfo": {
    "expectationsViolated": [
      "x1 is negative"
    ],
    "inputData": {
      "a": {"x1": 1,"y1": "a },
      "b": {
        "x2": 1,
        "y2": "aa"
      }
    },
    "outputRecord": {
      "x1": 1,
      "y1": "a",
      "x2": 1,
      "y2": "aa"
    },
    "missingInputData": false
  }
}
Multiple expectations
Use expect_all to specify multiple data quality constraints when records that fail validation should be included in the target dataset:

Python

Copy
@dlt.expect_all({"valid_count": "count > 0", "valid_current_page": "current_page_id IS NOT NULL AND current_page_title IS NOT NULL"})
Use expect_all_or_drop to specify multiple data quality constraints when records that fail validation should be dropped from the target dataset:

Python

Copy
@dlt.expect_all_or_drop({"valid_count": "count > 0", "valid_current_page": "current_page_id IS NOT NULL AND current_page_title IS NOT NULL"})
Use expect_all_or_fail to specify multiple data quality constraints when records that fail validation should halt pipeline execution:

Python

Copy
@dlt.expect_all_or_fail({"valid_count": "count > 0", "valid_current_page": "current_page_id IS NOT NULL AND current_page_title IS NOT NULL"})
You can also define constraints as a variable and pass it to one or more queries in your pipeline:

Python

Copy
valid_pages = {"valid_count": "count > 0", "valid_current_page": "current_page_id IS NOT NULL AND current_page_title IS NOT NULL"}

@expect_all(valid_pages)
@table
def raw_data():
  # Create raw dataset

@expect_all_or_drop(valid_pages)
@table
def prepared_data():
  # Create cleaned and prepared dataset
Create a development workflow with Delta Live Tables
You can create separate environments for your Delta Live Tables pipelines for development, staging, and production, allowing you to test and validate your transformation logic before you operationalize your pipelines. To use separate environments, create separate pipelines that target different databases, but use the same underlying code.

You can also decrease the cost of running tests by parameterizing your pipelines. For example, if you have the following query in a pipeline:

SQL

Copy
CREATE LIVE TABLE customer_list
AS SELECT * FROM sourceTable WHERE date > startDate
You can use this query in two different pipelines with separate configurations: a development pipeline that uses a subset of data for testing, and a production configuration that uses the complete dataset. The following example configurations use the startDate field to limit the development pipeline to a subset of the input data:

JSON

Copy
{
  "name": "Data Ingest - DEV",
  "target": "customers_dev",
  "libraries": [],
  "configuration": {
    "startDate": "2021-01-02"
  }
}
JSON

Copy
{
  "name": "Data Ingest - PROD",
  "target": "customers",
  "libraries": [],
  "configuration": {
    "startDate": "2010-01-02"
  }
}
Process data incrementally
Many applications require that tables be updated based on continually arriving data. However, as data sizes grow, the resources required to reprocess data with each update can become prohibitive. Delta Live Tables supports incremental computations to reduce the cost of ingesting new data and the latency at which new data is made available.

When an update is triggered for a pipeline, incremental tables process only new data that has arrived since the last update. Data already processed is automatically tracked by the Delta Live Tables runtime.

Incremental datasets with external data sources
You can define an incremental dataset by reading one or more inputs to your query as a stream. For example, you can read external data as a stream with the following code:

Python
Python

Copy
from pyspark.sql.types import StructType, StructField, StringType, TimestampType

inputPath = "/databricks-datasets/structured-streaming/events/"

jsonSchema = StructType([
    StructField("time", TimestampType(), True),
    StructField("action", StringType(), True)
])

@dlt.table
def streaming_bronze_table():
  return spark.read_stream.format('json').option('schema', jsonSchema).load(inputPath)
SQL
SQL

Copy
CREATE LIVE INCREMENTAL TABLE
AS SELECT
  to_timestamp(time) AS timestamp,
  action AS action
FROM json.`/databricks-datasets/structured-streaming/events/`
Incremental datasets within a pipeline
You can read incrementally from other tables in a pipeline:

Python
Python

Copy
@dlt.table
def streaming_silver_table:
  return read_stream("streaming_bronze_table").where(...)
SQL
SQL

Copy
CREATE INCREMENTAL LIVE TABLE streaming_silver_table
AS SELECT
  *
FROM
  STREAM(LIVE.streaming_bronze_table)
WHERE ...
Performing a full refresh
You might want to reprocess data that has already been ingested, for example, because you modified your queries based on new requirements or to fix a bug calculating a new column. You can reprocess data that’s already been ingested by instructing the Delta Live Tables system to perform a full refresh from the UI:

Go to the Pipeline Details page for your pipeline.
Click Blue Down Caret next to Start and select Full Refresh.
Mixing complete tables and incremental tables
You can mix different types of tables in a single pipeline. For example, the initial datasets in a pipeline, commonly referred to as bronze tables, often perform simple transformations. Reprocessing inefficient formats like JSON can be prohibitive with these simple transformations and are a perfect fit for incremental tables.

By contrast, the final tables in a pipeline, commonly referred to as gold tables, often require complicated aggregations that are not supported by Spark structured streaming. These transformations are better suited for materialization as a complete table.

By mixing the two types of tables into a single pipeline, you can avoid costly re-ingestion or re-processing of raw data and have the full power of Spark SQL to compute complex aggregations over an efficiently encoded and filtered dataset. The following example illustrates this type of mixed processing:

Python
Python

Copy
@dlt.view
def incremental_bronze():
  return (
    # Since this is a streaming source, this table is incremental.
    spark.readStream.format("cloudFiles")
      .option("cloudFiles.format", "json")
      .load("abfss://path/to/raw/data")
  )

@dlt.table
def incremental_silver():
  # Since we read the bronze table as a stream, this silver table is also
  # updated incrementally.
  return dlt.read_stream("incremental_bronze").where(...)

@dlt.table
def complete_gold():
  # This table will be recomputed completely by reading the whole silver table
  # when it is updated.
  return dlt.read("incremental_silver").groupBy("user_id").count()
SQL
SQL

Copy
CREATE INCREMENTAL LIVE VIEW incremental_bronze
  AS SELECT * FROM cloud_files(
    "s3://path/to/raw/data", "json")

CREATE LIVE TABLE incremental_silver
  AS SELECT * FROM LIVE.incremental_bronze WHERE...

CREATE LIVE TABLE complete_gold
  AS SELECT count(*) FROM LIVE.incremental_silver GROUP BY user_id
Learn more about using Auto Loader to efficiently read JSON files from Azure storage for incremental processing.

Incremental joins
Delta Live Tables supports various join strategies for updating tables.

Stream-batch joins
Stream-batch joins are a good choice when denormalizing a continuous stream of data with a primarily static dimension table. Each time the derived dataset is updated, new records from the stream are joined with a static snapshot of the batch table when the update started. Records added or updated in the static table are not reflected in the table until a full refresh is performed.

The following are examples of stream-batch joins:

Python
Python

Copy
@dlt.table
def customer_sales():
  return dlt.read_stream("sales").join(read("customers"), ["customer_id"], "left")
SQL
SQL

Copy
CREATE INCREMENTAL LIVE TABLE customer_sales
AS SELECT * FROM STREAM(LIVE.sales)
  INNER JOIN LEFT READ LIVE.customers ON customer_id
In continuous pipelines, the batch side of the join is regularly polled for updates with each micro-batch.

Incremental aggregation
Simple distributive aggregates like count, min, max, or sum, and algebraic aggregates like average or standard deviation can also be calculated incrementally. Databricks recommends incremental aggregation for queries with a limited number of groups, for example, a query with a GROUP BY country clause.

Only new input data is read with each update, but the underlying Delta table is completely overwritten.

https://docs.microsoft.com/en-us/azure/databricks/data-engineering/delta-live-tables/delta-live-tables-user-guide
