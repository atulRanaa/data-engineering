# Deep Dive: Apache Airflow

If data is the raw material, and processing engines (Spark) are the factories, **Apache Airflow** is the logistics manager that ensures the trucks arrive on time. Airflow is the industry standard for workflow scheduling and orchestration.

---

## 1. History & Origins

In 2014, Maxime Beauchemin (also the creator of Apache Superset) was working as a Data Engineer at Airbnb. Airbnb's data team was drowning in a complex web of cron jobs, shell scripts, and unmanageable dependencies. When a Hadoop job failed overnight, no one knew which downstream MySQL scripts should be paused.

Maxime created Airflow to solve this by treating **Configuration as Code**. Instead of defining workflows in a drag-and-drop XML GUI (like Informatica or Talend), he designed Airflow so data engineers could write native Python code to define the entire pipeline structure. Airflow was open-sourced in 2015 and quickly became the foundational tool for modern Data Engineering orchestration.

---

## 2. Core Architecture and Internal Insights

Airflow does not actually *process* data. It simply holds the blueprint for the pipeline and sends commands to other systems (like sending a SQL string to Snowflake or launching a Spark cluster on AWS).

### The Directed Acyclic Graph (DAG)
A pipeline in Airflow is represented as a DAG.
- **Directed:** The tasks have a strict order (A runs before B).
- **Acyclic:** The pipeline cannot loop back on itself (B cannot trigger A). It must flow in one direction toward completion.
- **Graph:** The network of tasks and their dependencies.

**Internal Insight: The Scheduler vs. The Executor**
Airflow's architecture is strictly decoupled.
The **Scheduler** is a Python daemon that constantly scans the `dags/` folder. It looks at the clock, realizes a DAG is scheduled to run at 2:00 AM, and puts the tasks into a message queue (like Redis or RabbitMQ).
The **Executor** is the actual worker. A small company might use a `LocalExecutor` (running tasks on the same server), while a massive enterprise uses a `KubernetesExecutor` (spinning up a brand new Docker container for every single task, ensuring total isolation).

---

## 3. Hands-on Code Example

Writing a DAG in Python allows you to use `for` loops, variables, and imported libraries to dynamically generate your pipeline structure.

```python
from airflow import DAG
from airflow.providers.snowflake.operators.snowflake import SnowflakeOperator
from airflow.providers.amazon.aws.sensors.s3 import S3KeySensor
from datetime import datetime, timedelta

# 1. Define Default Arguments for retries and timeouts
default_args = {
    'owner': 'data_engineering_team',
    'depends_on_past': False,
    'email_on_failure': True,
    'email': ['alerts@company.com'],
    'retries': 3,
    'retry_delay': timedelta(minutes=5),
}

# 2. Instantiate the DAG object
with DAG(
    dag_id='daily_sales_etl',
    default_args=default_args,
    description='Extract sales from S3 and load to Snowflake',
    schedule_interval='@daily',
    start_date=datetime(2024, 1, 1),
    catchup=False  # Do not backfill previous dates on initial deployment
) as dag:

    # 3. Define Tasks (Operators & Sensors)
    
    # Task A: A Sensor that waits for a specific file to physically appear in AWS S3
    wait_for_file = S3KeySensor(
        task_id='wait_for_sales_csv',
        bucket_key='s3://company-raw-data/sales/{{ ds }}/transactions.csv',
        aws_conn_id='aws_default'
    )

    # Task B: An Operator that executes a SQL string inside the Snowflake Warehouse
    load_to_snowflake = SnowflakeOperator(
        task_id='copy_into_snowflake',
        snowflake_conn_id='snowflake_default',
        sql="""
            COPY INTO raw.sales_transactions 
            FROM @my_s3_stage/sales/{{ ds }}/
            FILE_FORMAT = (TYPE = CSV);
        """
    )
    
    # Task C: Another SQL command to trigger dbt or a transform
    transform_data = SnowflakeOperator(
        task_id='build_reporting_tables',
        snowflake_conn_id='snowflake_default',
        sql="CALL build_daily_sales_aggregates();"
    )

    # 4. Set Dependencies (The Graph Structure!)
    # The file must exist before we COPY, and we must COPY before we Transform
    wait_for_file >> load_to_snowflake >> transform_data
```

Notice the use of **Jinja Templating** (`{{ ds }}`). When the DAG runs for Jan 5th, Airflow automatically renders `{{ ds }}` as `2024-01-05`, ensuring the pipeline is entirely date-agnostic.

---

!!! abstract "References & Foundational Reading"
    - **Designing Data-Intensive Applications:** Chapter 10 (Batch Processing) covers the necessity of DAGs for coordinating multi-step MapReduce or Spark jobs.
    - **Airflow Documentation:** *Concepts - Architecture* regarding the decoupling of Schedulers and Executors.
