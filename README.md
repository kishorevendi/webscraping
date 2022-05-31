# Data Quality @ Vodafone
Data quality can affect all of us, whether determining the accuracy of customer data for sales, marketing or billing. The right quality data supports each of our teams in making optimal commercial decisions.

Some use cases involve a small amount of data, though with the potentially great (adverse) commercial impact. Other use cases involve a large ammount of data also with potentially great commercial impact. Nevertheless, one truth unites use cases. Data quality is a persistent challenge that MUST be managed.

As a persistent challenge, we must look at efficient (yet flexible) means of mastering it. We want the following principles for our own data quality approach:

* a modular approach that can plug-in to other (types of) data pipeline(s)
* an integral set of metrics which can be output to Vodafone approved public cloud datastores (e.g. GCS, BigQuery)
* an integral set of metrics which monitor processing time (cost) for the module
* appropriate (downstream) propagation of defined data quality metrics to downstream steps in a (data) pipeline
* a standards-based metrics structure enabling the use of visualisation tools, alongside post-processing and anomaly detection techniques
* a process for submitting (rather translating) data quality metrics requests from data domain experts (possibly pseudo-SQL?)
* (TBC) a lineage which allows for the tracking of metrics through a pipeline (optional)

Efficient processing should be engineered. Invocation of this efficient processing can be configured and is an approach that has achieved a degree of success amongst the big data community at Vodafone. How the configuration works remains open for discussion at this stage (27/11/2020).

## Directory Structure for THIS Repository
The repository is designed initially for code sharing and collaboration from local markets and group.

1. PLEASE use the [Local Market](https://github.vodafone.com/vfgroup-tsa-datagovernance/Data-Quality/tree/master/Local%20Market) folder to contribute to the data quality discussion.
2. MAIN | PLATFORM | MODULE subfolder will appear under the repository root to bring together the main components of the data quality module.

For further information or questions you have the following resources at your disposal:

* [Data Quality Forum @ Microsoft Teams](https://teams.microsoft.com/l/team/19%3a2dee42f768394cd0b68e015876413089%40thread.tacv2/conversations?groupId=131374e6-fda7-4e59-b333-b3ab0335b4c4&tenantId=68283f3b-8487-4c86-adb3-a5228f18b893), our active discussion forum
* [Data Quality JIRA Project](https://jira.sp.vodafone.com/plugins/servlet/project-config/DQF/summary), GitHub smart commits are enabled.
* [Data Quality Framework Documentation @ Confluence](https://confluence.sp.vodafone.com/x/j5B0Bw), our detailed documentation and design principles.
* Here is the detailed documentation and help guide for DQ Tool on Confleunce:
    * [DQ Tool : MVP](https://confluence.sp.vodafone.com/display/ACE/Data+Quality+-+MVP)
    * [Data Quality Tooling Strategy](https://confluence.sp.vodafone.com/display/ACE/Data+Quality+Tooling+Strategy)
    * [DQ Tool Architecture](https://confluence.sp.vodafone.com/display/ACE/DQ+Architecture+and+Technical+Requirements)
    * [DQ Tool Input parameter validation](https://confluence.sp.vodafone.com/display/ACE/Input+parameter+validation) : Guide for validations on input parameters.
    * [DQ Tool Predefined Rules](https://confluence.sp.vodafone.com/display/ACE/Pre-defined+SQL+Rules) : Guide for each of the predefined rules.
* Latest DQ Tool jar is uploaded in MS Teams channel "DQ Tool". Contact ACoE Data Governance (vicente.rustarazohervas@vodafone.com or atul.ruparelia@vodafone.com) for access.

***Note** : Refer to document **'DQ_Tool_Guide.docx'** for step-by-step guidance on how to configure the tool on a new environment on GCP.*


# Getting started
### Assembly
Once you have cloned your repository you can compile with SBT
> sbt assembly

or

> assembly

### Dataproc deployment
Create standard multinode dataproc cluster:
https://cloud.google.com/dataproc/docs/guides/create-cluster
When creating the cluster you must set this mandatory properties to enable logs
> "dataproc:dataproc.logging.stackdriver.enable":"true" \
> "dataproc:dataproc.logging.stackdriver.job.driver.enable": "true" \
> "dataproc:dataproc.logging.stackdriver.job.yarn.container.enable": "true"

### Input parameters config file json
Needs to set the JSON config file with the required rules and format:
https://confluence.sp.vodafone.com/display/ACE/Pre-defined+SQL+Rules
Save the JSON config file in path with read permissions for Dataproc Jobs

### Enable APIs
You must enable cloud logging API to access Standard output logs from DQ tool
https://console.cloud.google.com/marketplace/product/google/logging.googleapis.com?q=search&referrer=search&cloudshell=false&orgonly=true&supportedpurview=organizationId

### Run job
List of parameters to run the job in dataproc:
>gcloud dataproc jobs submit spark \
>       --project your-gcp-project-name \
>       --cluster your-dataproc-cluster-name \
>       --region your-gcp-region \
>       --class vodafone.dataquality.app.DataQualityCheckApp  \
>       --jars path/to/your/compiled.jar \
>       --properties="spark.submit.deployMode=cluster" \
>       -- --bucket your-gcs-bucket-name --jsonFile path/to/your/configfile.json --mode cluster --bqTempBucket your-bigquery-sink-gcs-bucket --bqDataset your-bigquery-sink-dataset --bqProject your-gcp-project-name --bqParentProject your-gcp-project-name --bqLogTable your-bigquery-sink-tablename

Add these optional parameters to the arguments line for time windowing and partition, column dates must be set on JSON config file previously.
>--format yyyy/MM/dd
>--startDate window/start/date/set/in/format
>--endDate window/end/date/set/in/format

Add these optional parameters to the arguments line if using source name parameterization. The corresponding source name parameters must be specified in JSON config file.
>--sourcePathKV k1=path/to/your/file1/, k2=path/to/your/file2/

### Logging
You can explore the logs on logs explorer: https://console.cloud.google.com/logs/query
With the following query to filter outputs for your dataproc job
>resource.type="cloud_dataproc_job"
>resource.labels.job_id="yourdataprocjobid"

### BigQuery logs
The output sink for bigquery will be loaded into the table set on input parameters. One column "logrow" contains all the info for each rule.

This query to trim the json format in a columnar view:
>select  TRIM(JSON_EXTRACT(logrow, '$.UniqueId'),'"') AS UniqueId,
TIMESTAMP(TRIM(JSON_EXTRACT(logrow, '$.SubmissionDateTime'),'"')) AS SubmissionDateTime,
case when TRIM(JSON_EXTRACT(logrow, '$.Status')) like "%SUCCESS%" then "Success" else "Failed" end AS Status,
TRIM(JSON_EXTRACT(logrow, '$.MetricId'),'"') AS MetricRuleType,
TRIM(JSON_EXTRACT(logrow, '$.Rule'),'"') AS MetricRuleValue,
TRIM(JSON_EXTRACT(logrow, '$.MetricResult'),'"') AS MetricResult,
TRIM(JSON_EXTRACT(logrow, '$.SQLQuery'),'"') AS SQLQuery,
TRIM(JSON_EXTRACT(logrow, '$.SourcePath'),'"') AS SourcePath,
TRIM(JSON_EXTRACT(logrow, '$.SourceType'),'"') AS SourceType,
TRIM(JSON_EXTRACT(logrow, '$.StartDate'),'"') AS StartDate,
TRIM(JSON_EXTRACT(logrow, '$.EndDate'),'"') AS EndDate,
TRIM(JSON_EXTRACT(logrow, '$.Status'),'"') AS Status_details,
from `your-gcp-project-name.your-bigquery-sink-dataset.your-bigquery-sink-tablename`
order by SubmissionDateTime desc

Alternative is create a bigquery sink from Logs router:
https://console.cloud.google.com/logs/router

### Monitoring metrics
It's possible to create metrics for monitoring purposes:
https://console.cloud.google.com/logs/metrics

Apart from standard count metrics, based on number of rows that matches a filter you can create advanced distribution metrics to extract numeric values from the output.
Given a log filter
>resource.type= "cloud_dataproc_job"
jsonPayload.container_logname = "stdout"
jsonPayload.message:"ALPHANUMERICALMETRICID"

ALPHANUMERICALMETRICID is an additional row that contains ID of the rule and result of the rule concatenated as example:
> NBIDIBIDABD85

Where NBIDIBIDABD is the id of the rule and 85 is the result of the rule. Given the previous filter to extract the numeric result in the rule we can use a regexp:
> ([0-9]+)

With the metric created you can look at monitor in real time:
https://console.cloud.google.com/monitoring/metrics-explorer

### Monitoring alerting

With the metrics you can create alerts based on custom tresholds:
https://cloud.google.com/logging/docs/logs-based-metrics/charts-and-alerts

### BI reports & Data Studio
### Orchestration

With cloud composer (Apache Airflow) it's possible cluster deployment and job automation.
First you need to enable cloud composer api:
https://console.cloud.google.com/marketplace/product/google/composer.googleapis.com

Needs to create and environment and a DAG:
https://cloud.google.com/composer/docs/quickstart

This is the DAG template code:
```
from datetime import timedelta, datetime
from airflow import models
from airflow.operators.bash_operator import BashOperator
from airflow.contrib.operators.dataproc_operator import DataprocClusterCreateOperator,DataprocClusterDeleteOperator,DataProcSparkOperator
from airflow.utils.trigger_rule import TriggerRule

yesterday = datetime(2021, 5, 31)

default_dag_args = {
'start_date': yesterday,
'depends_on_past': False,
'email_on_failure': False,
'email_on_retry': False,
'retries': 1,
'retry_delay': timedelta(minutes=5)
}

with models.DAG(
'DQ_Auto_Execution',
description='DQ Automation',
schedule_interval=timedelta(days=1),
default_args=default_dag_args) as dag:
    start = BashOperator(
        task_id='Header',
        bash_command='echo "DQ Execution Started"'
    )

    create_cluster = DataprocClusterCreateOperator(
        task_id='create_dataproc_cluster',
        # ds_nodash is an airflow macro for "[Execution] Date string no dashes"
        # in YYYYMMDD format. See docs https://airflow.apache.org/code.html?highlight=macros#macros
        cluster_name='cluster-dq-{{ ds_nodash }}',
        num_workers=2,
        master_machine_type='n1-standard-4',
        master_disk_size=1024,
        worker_machine_type='n1-standard-4',
        worker_disk_size=1024,
        properties={"dataproc:dataproc.logging.stackdriver.enable":"true","dataproc:dataproc.logging.stackdriver.job.driver.enable": "true","dataproc:dataproc.logging.stackdriver.job.yarn.container.enable": "true"},
        zone='your-zone',
        ##network_uri='your-vpc-net',
        subnetwork_uri='your-vpc-subnet',
        region='your-region',
        tags='allow-ssh',
        project_id='your-gcp-project-name')

    call_DQF = DataProcSparkOperator(
        task_id ='execute_spark_job_cluster_test',
        dataproc_spark_jars='your/path/to/your/compiled.jar',
        cluster_name='cluster-dq-{{ ds_nodash }}',
        main_class = 'vodafone.dataquality.app.DataQualityCheckApp',
        project_id='your-gcp-project-name',
        region='your-region',
        dataproc_spark_properties={'spark.submit.deployMode':'cluster'},
        arguments=['--bucket','your-gcs-bucket-name','--jsonFile','your/path/to/your/configfile.json','--mode','cluster', '--bqTempBucket' , 'your-bigquery-sink-gcs-bucket' , '--bqDataset' , 'your-bigquery-sink-dataset' , '--bqProject' , 'your-gcp-project-name' , '--bqParentProject' , 'your-gcp-project-name'  '--bqLogTable' , 'your-bigquery-sink-tablename']
   )
       # Delete the Cloud Dataproc cluster.
    delete_cluster = DataprocClusterDeleteOperator(
        task_id='delete_dataproc_cluster',
        cluster_name='cluster-dq-{{ ds_nodash }}',
        region='your-region',
        project_id='your-gcp-project-name',
        # This will tear down the cluster even if there are failures in upstream tasks.
        trigger_rule=TriggerRule.ALL_DONE)

    end = BashOperator(
        task_id='Footer',
        bash_command='echo "DQ Execution Done"'
    )

start >> create_cluster >> call_DQF >> delete_cluster >> end
```

<b>Note:</b>

* If you want the DQ tool to run sql queries for a given time interval i.e. filter out results based on start date and end date, then you can specify it in runtime arguments in DAG along with the format in which the date is entered. By default, start and end dates are optional and if not specified, it takes all records from the dataset into consideration to generate output.
  ```
  call_DQF = DataProcSparkOperator(
        task_id ='execute_spark_job_cluster_test',
        dataproc_spark_jars='your/path/to/your/compiled.jar',
        cluster_name='cluster-dq-{{ ds_nodash }}',
        main_class = 'vodafone.dataquality.app.DataQualityCheckApp',
        project_id='your-gcp-project-name',
        region='your-region',
        dataproc_spark_properties={'spark.submit.deployMode':'cluster'},
        arguments=['--bucket','your-gcs-bucket-name','--jsonFile','your/path/to/your/configfile.json','--mode','cluster', '--bqTempBucket' , 'your-bigquery-sink-gcs-bucket' , '--bqDataset' , 'your-bigquery-sink-dataset' , '--bqProject' , 'your-gcp-project-name' , '--bqParentProject' , 'your-gcp-project-name'  '--bqLogTable' , 'your-bigquery-sink-tablename', '--format', 'yyyy/MM/dd', '--startDate' , 'window/start/date/set/in/format', '--endDate', 'window/end/date/set/in/format']
   )
   ```

* If any of your datafiles are in AVRO format, then you need to add AVRO jar file in GCS bucket and AVRO package name and jar filepath  in the DAG. Below is the modified call to DQ Tool. Replace this in the DAG above.
    ```
    call_DQF = DataProcSparkOperator(
        task_id ='execute_spark_job_cluster_test',
        dataproc_spark_jars=['your/path/to/your/compiled.jar', 'your/path/to/AVRO/jar-file.jar'],
        cluster_name='cluster-dq-{{ ds_nodash }}',
        main_class = 'vodafone.dataquality.app.DataQualityCheckApp',
        project_id='your-gcp-project-name',
        region='your-region',
        dataproc_spark_properties={'spark.submit.deployMode':'cluster', 'packages':'org.apache.spark:spark-avro_2.12:3.1.1'},
        arguments=['--bucket','your-gcs-bucket-name','--jsonFile','your/path/to/your/configfile.json','--mode','cluster', '--bqTempBucket' , 'your-bigquery-sink-gcs-bucket' , '--bqDataset' , 'your-bigquery-sink-dataset' , '--bqProject' , 'your-gcp-project-name' , '--bqParentProject' , 'your-gcp-project-name'  '--bqLogTable' , 'your-bigquery-sink-tablename']
   )
    ```

* If you want to parameterize the source path of any file, use optional parameter --sourcePathKV. For eg : the source path keeps changing everyday because date is appended in the folder name. In this case, we pass the complete path as key-value pair as one of the parameters in yaml file. The key is passed as source path in config json. Replace this in the DAG above.
    ```
    call_DQF = DataProcSparkOperator(
        task_id ='execute_spark_job_cluster_test',
        dataproc_spark_jars=['your/path/to/your/compiled.jar', 'your/path/to/AVRO/jar-file.jar'],
        cluster_name='cluster-dq-{{ ds_nodash }}',
        main_class = 'vodafone.dataquality.app.DataQualityCheckApp',
        project_id='your-gcp-project-name',
        region='your-region',
        dataproc_spark_properties={'spark.submit.deployMode':'cluster', 'packages':'org.apache.spark:spark-avro_2.12:3.1.1'},
        arguments=['--bucket','your-gcs-bucket-name','--jsonFile','your/path/to/your/configfile.json','--mode','cluster', '--bqTempBucket' , 'your-bigquery-sink-gcs-bucket' , '--bqDataset' , 'your-bigquery-sink-dataset' , '--bqProject' , 'your-gcp-project-name' , '--bqParentProject' , 'your-gcp-project-name'  '--bqLogTable' , 'your-bigquery-sink-tablename', '--sourcePathKV' , 'k1=path/to/file/']
   )
    ```
   


## Things to keep in mind while writing rules
Here are some key points which can be kept in mind while writing rules in a config file. It is helpful in cases when you are using the Excel UI utlity or writing the rules by yourself in a json file. It will help to generate a config file in correct format to be fed to DQ tool. 

* The "sourceName" which acts as a view name for a dataset should only consist of alphabets (lower or upper) and/or numbers. Special characters are not recognised when creating a tempory view using this source name. Even using escape character \ before a special character will not help. 
    * Valid sourceName : book, Book, BOOK, Book123, etc.
    * Invalid sourceNames : book$, book.s, book\\$, book\\", etc. 
* When using rule type "SourceColumnAccuracy", remember to seperate all column names in ruleCoumnName only by a comma. No other character like semicolon, colon, etc. is supported. Spaces are permitted between column names and comma for readability purposes.
    * Valid value : "id,name,email" or "id , name , email"
    * Invalid value : "id ; name ; email"
* Remember that matching for column names entered in rule "SourceColumnAccuracy" is case sensitive. This means that "name" and "Name" are considered as different columns.
* When writing your own sql query or specifying a filter for sql query in ruleValue, if double quotes (" ") are needed to specify some string value, then remember to use an escape character (\\\) before the double quotes to ensure that the quotes are a part of rule value and different from the double quotes which mark the start and end of rule value in json format.
    * Example : "ruleValue" : "select count(\*) from table where column1 in (\\\"Male\\\")"
* For rules like MinRule, MaxRule, SumRule, NullCheck and NotNullCheck, setting "allowNulls" field to Yes or No makes no sense at all. Example : allowing nulls to get max value of a column will not make a difference in output or for checking nulls in a rule, setting allowNulls as No will be conflicting. Remember to leave that field blank for these rules.
* The rule RowCountProportion calculates the proportion of expected rows and actual rows. It is advisable to enter a positive integer in "ruleValue" key to make the output a reasonable proportion. 
    * Example, advisable "ruleValue" key values : 10, 40, etc. If you enter -10 or 10.5, DQ Tool will work and return the output but it is not reasonable to expect -10 rows or 10.5 rows in a dataset.
* There are some predefined rules which apply on metadata of dataset like ColumnToBeOfType, SourceColumnCount, SourceNameAccuracy and SourceColumnsAccuracy. For such rules, leave keys "periodColumnName" and "allowNulls" as blank.
* There are multiple ways of defining a sourcePath in config file:
    * Complete path with filename : this/is/the/path/filename.extension
    * Complete path until directory : this/is/the/path/ (The sql query will run on all files within this directory)
    * Path matching : this/is/the/path*/, this/is/the/\*/\*, this/is/the/path/\* ,this/is/the/path/\*.extension, this/is/the/path/file\*.extension, etc.
* To define complex sql queries with nested date filter, use keyword "addPeriodColumn".
    * Example : If you want to execute this query => "select count(\*) from table where (column1 > 5 and date between startDate and endDate) or column2 > 10"
    * To fit such a date filter at a specified location, write this in ruleValue => "select count(\*) from table where (column1 > 5 and <b>addPeriodColumn</b>) or column2 > 10"
    * Specify the date column appropriately in "periodColumnName" and the required sql is generated by the tool.


## Unit Tests
Unit test cases are written using library FunSuite. A simple example for writing and running a test case is illustrated here: https://docs.scala-lang.org/getting-started/intellij-track/testing-scala-in-intellij-with-scalatest.html

### Writing a test case

'test' is function that comes from the FunSuite library that collects results from assertions within the function body.

Syntax to define a test function
> test(testName = "testing functionality for XYZ")

Syntax to ignore a test case for time-being
> ignore(testName = "testing functionality for XYZ")

### Assertions in test case
'assert' takes a boolean condition and determines whether the test passes or fails.
Syntax for using assert:
> assert(expected_value === actual_value)

If assert returns true, the test passes, else it fails.

### Running a test case
* To run a particular test case => right-click on or within the test case body and select run.
* To run all test cases within a scala test => right-click on the class name in the project window and select run.
* To run all test cases for all scala classes within a folder => right-click on the folder in the project window and select run.

### View test case results
After running the required unit tests, you can view the test results in the 'Run' window. It shows a count of all the tests that passed, failed or were ignored. Below is shown the visual representation for each test case output
* Passed test cases : :heavy_check_mark: .
* Failed test cases : :heavy_exclamation_mark:.
* Ignored test cases : ~~:green_circle:~~
