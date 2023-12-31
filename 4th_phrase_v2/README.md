<h1>Module 3: Orchestrated batch / stream transformations using dbt + Airflow with Feast (Snowflake)</h1>

> **Note:** This module is still WIP, and does not have a public data set to use. There is a smaller dataset visible in `data/`

This is a very similar module to module 1. The key difference is now we'll be using a data warehouse (Snowflake) in combination with dbt + Airflow to ensure that batch features are regularly generated. 

**Caveats**
- Feast does not itself handle orchestration of data pipelines (transforming features, materialization) and relies on the user to configure this with tools like dbt and Airflow.
- Feast does not ensure consistency in transformation logic between batch and stream features

**Architecture**
- **Data sources**: Snowflake
- **Online store**: Redis
- **Orchestrator**: Airflow + dbt
- **Use case**: Fraud detection

<img src="architecture.png" width=750>

<h2>Table of Contents</h2>

- [Workshop](#workshop)
  - [Step 1: Install Feast](#step-1-install-feast)
  - [Step 2: Inspect the `feature_store.yaml`](#step-2-inspect-the-feature_storeyaml)
  - [Step 3: Spin up services (Kafka + Redis + Feast SQL Registry + Feast services)](#step-3-spin-up-services-kafka--redis--feast-sql-registry--feast-services)
  - [Step 4: Set up dbt models for batch transformations](#step-4-set-up-dbt-models-for-batch-transformations)
  - [Step 5: Run `feast apply`](#step-5-run-feast-apply)
  - [Step 6: Set up orchestration](#step-6-set-up-orchestration)
    - [Step 6a: Setting up Airflow to work with dbt](#step-6a-setting-up-airflow-to-work-with-dbt)
    - [Step 6b: Examine the Airflow DAG](#step-6b-examine-the-airflow-dag)
    - [6c: Enable the DAG](#6c-enable-the-dag)
      - [Q: What if different feature views have different freshness requirements?](#q-what-if-different-feature-views-have-different-freshness-requirements)
    - [Step 6d (optional): Run a backfill](#step-6d-optional-run-a-backfill)
  - [Step 7: Retrieve features + test stream ingestion](#step-7-retrieve-features--test-stream-ingestion)
    - [Overview](#overview)
    - [Time to run code!](#time-to-run-code)
  - [Step 8: Options for orchestrating streaming pipelines](#step-8-options-for-orchestrating-streaming-pipelines)
- [Conclusion](#conclusion)
  - [Limitations](#limitations)
  - [Why Feast?](#why-feast)
- [FAQ](#faq)
    - [How do I iterate on features?](#how-do-i-iterate-on-features)
    - [How does this work in production?](#how-does-this-work-in-production)

# Workshop
## Step 1: Install Feast

First, we install Feast with Snowflake and Postgres and Redis support:
```bash
pip install "feast[snowflake,postgres,redis]"
```

## Step 2: Inspect the `feature_store.yaml`

```yaml
project: feast_demo_local
provider: local
registry:
  registry_type: sql
  path: postgresql://postgres:mysecretpassword@127.0.0.1:55001/feast
online_store:
  type: redis
  connection_string: localhost:6379
offline_store:
  type: snowflake.offline
  account: ${SNOWFLAKE_DEPLOYMENT_URL}
  user: ${SNOWFLAKE_USER}
  password: ${SNOWFLAKE_PASSWORD}
  role: ${SNOWFLAKE_ROLE}
  warehouse: ${SNOWFLAKE_WAREHOUSE}
  database: TECTON_DEMO_DATA
  schema: FRAUD
entity_key_serialization_version: 2
```

##  Step 3: Spin up services (Kafka + Redis + Feast SQL Registry + Feast services)

We use Docker Compose to spin up the services we need.
- This deploys an instance of Redis, Postgres for a registry, a Feast feature server + push server.
- This also uses `transactions.parquet` to generate streaming feature values to ingest into the online store with dummy timestamps

Start up the Docker daemon and then use Docker Compose to spin up the services as described above:
- You may need to run `sudo docker-compose up` if you run into a Docker permission denied error
```console
$ docker-compose up

Creating network "module_3_default" with the default driver
Creating registry  ... done
Creating zookeeper ... done
Creating redis     ... done
Creating broker    ... done
Creating tx_kafka_events ... done
Creating feast_feature_server ... done
Attaching to zookeeper, redis, registry, broker, kafka_events, feast_feature_server
...
```

## Step 4: Set up dbt models for batch transformations
> **TODO(adchia):** Generate parquet file to upload for public Snowflake dataset for features
 
There's already a dbt model that generates batch transformations. You just need to init this:

> **Note:** You'll need to install dbt-snowflake as well! `brew tap dbt-labs/dbt` and `brew install dbt-snowflake`

To initialize dbt with your own credentials, do this
```bash
cd dbt/feast_demo; dbt init; dbt run
```

This will create the initial tables we need for Feast

## Step 5: Run `feast apply`
In this example, we're using a test database in Snowflake. 

To get started, go ahead and register the feature repository
```console
<!-- 
Note: first you need to export environment variables 
matching the above variables:

export SNOWFLAKE_DEPLOYMENT_URL="[YOUR DEPLOYMENT]
export SNOWFLAKE_USER="[YOUR USER]
export SNOWFLAKE_PASSWORD="[YOUR PASSWORD]
export SNOWFLAKE_ROLE="[YOUR ROLE]
export SNOWFLAKE_WAREHOUSE="[YOUR WAREHOUSE]
export SNOWFLAKE_DATABASE="[YOUR DATABASE]
-->
$ cd feature_repo; feast apply

Created entity user
Created feature view aggregate_transactions_features
Created feature view credit_scores_features
Created feature service model_v1
Created feature service model_v2

Deploying infrastructure for aggregate_transactions_features
Deploying infrastructure for credit_scores_features
```
## Step 6: Set up orchestration
### Step 6a: Setting up Airflow to work with dbt

We setup a standalone version of Airflow to set up the `PythonOperator` (Airflow now prefers @task for this) and `BashOperator` which will run incremental dbt models. We use dbt to define batch transformations from Snowflake, and once the incremental model is tested / ran, we run materialization.

The below script will copy the dbt DAGs over. In production, you'd want to use Airflow to sync with version controlled dbt DAGS (e.g. that are sync'd to S3).

```bash
# If not already done, export Snowflake related environment variables used above:
# export SNOWFLAKE_DEPLOYMENT_URL="[YOUR DEPLOYMENT]
# export SNOWFLAKE_USER="[YOUR USER]
# export SNOWFLAKE_PASSWORD="[YOUR PASSWORD]
# export SNOWFLAKE_ROLE="[YOUR ROLE]
# export SNOWFLAKE_WAREHOUSE="[YOUR WAREHOUSE]
# export SNOWFLAKE_DATABASE="[YOUR DATABASE]
cd ../airflow_demo; sh setup_airflow.sh
```

### Step 6b: Examine the Airflow DAG

The example dag is going to run on a daily basis and materialize *all* feature views based on the start and end interval. Note that there is a 1 hr overlap in the start time to account for potential late arriving data in the offline store. 

With dbt incremental models, the model itself in incremental mode selects overlapping windows of data to account for late arriving data. Feast materialization similarly has a late arriving threshold.

```python
with DAG(
    dag_id='feature_dag',
    start_date=pendulum.datetime(2021, 1, 1, tz="UTC"),
    description='A dbt + Feast DAG',
    schedule="@daily",
    catchup=False,
    tags=["feast"],
) as dag:
    dbt_test = BashOperator(
        task_id="dbt_test",
        bash_command="""
            cd ${AIRFLOW_HOME}; dbt test --models "aggregate_transaction_features"
            """,
        dag=dag,
    )
    
    dbt_run = BashOperator(
        task_id="dbt_run",
        bash_command="""
            cd ${AIRFLOW_HOME}; dbt run --models "aggregate_transaction_features"
            """,
        dag=dag,
    )
    
    @task()
    def materialize(data_interval_start=None, data_interval_end=None):
        repo_config = RepoConfig(
            registry=RegistryConfig(
            registry_type="sql",
            path="postgresql://postgres:mysecretpassword@127.0.0.1:55001/feast",
        ),
        project="feast_demo_local",
        provider="local",
        offline_store=SnowflakeOfflineStoreConfig(
            account=Variable.get("SNOWFLAKE_DEPLOYMENT_URL"),
            user=Variable.get("SNOWFLAKE_USER"),
            password=Variable.get("SNOWFLAKE_PASSWORD"),
            role=Variable.get("SNOWFLAKE_ROLE"),
            warehouse=Variable.get("SNOWFLAKE_WAREHOUSE"),
            database=Variable.get("SNOWFLAKE_DATABASE"),
            schema_=Variable.get("SNOWFLAKE_SCHEMA"),
        ),
        online_store=RedisOnlineStoreConfig(connection_string="localhost:6379"),
        entity_key_serialization_version=2
      )
      store = FeatureStore(config=repo_config)
      store.materialize(data_interval_start.subtract(hours=1), data_interval_end)
    
    # Setup DAG
    dbt_test >> dbt_run >> materialize()
```

### 6c: Enable the DAG
Now go to `localhost:8080`, use Airflow's auto-generated admin password to login, and toggle on the DAG. It should run one task automatically. After waiting for a run to finish, you'll see a successful job:

![](airflow.png)

#### Q: What if different feature views have different freshness requirements?

There's no built in mechanism for this, but you could store this logic in the feature view tags (e.g. a `batch_schedule`).
 
Then, you can parse these feature view in your Airflow job. You could for example have one DAG that runs all the daily `batch_schedule` feature views, and another DAG that runs all feature views with an hourly `batch_schedule`.

### Step 6d (optional): Run a backfill
To run a backfill (i.e. process previous days of the above while letting Airflow manage state), you can do (from the `airflow_demo` directory):

> **Warning:** This works correctly with the Redis online store because it conditionally writes. This logic has not been implemented for other online stores yet, and so can result in incorrect behavior

```bash
export AIRFLOW_HOME=$(pwd)/airflow_home
airflow dags backfill \
    --start-date 2021-07-01 \
    --end-date 2021-07-15 \
    feature_dag
```

## Step 7: Retrieve features + test stream ingestion
### Overview
Feast exposes a `get_historical_features` method to generate training data / run batch scoring and `get_online_features` method to power model serving.

To achieve fresher features, one might consider using streaming compute.There are two broad approaches with streaming
1. **[Simple, semi-fresh features]** Use data warehouse / data lake specific streaming ingest of raw data.
   - This means that Feast only needs to know about a "batch feature" because the assumption is those batch features are sufficiently fresh.
   - **BUT** there are limits to how fresh your features are. You won't be able to get to minute level freshness.
2. **[Complex, very fresh features]** Build separate streaming pipelines for very fresh features
   - It is on you to build out a separate streaming pipeline (e.g. using Spark Structured Streaming or Flink), ensuring the transformation logic is consistent with batch transformations, and calling the push API as per [module 1](../module_1/README.md). 

Feast will help enforce a consistent schema across batch + streaming features as they land in the online store. 

### Time to run code!
Now, Run [Jupyter notebook](feature_repo/module_3.ipynb)

## Step 8: Options for orchestrating streaming pipelines
We don't showcase how this works, but broadly there are many approaches to this. In all the approaches, you'll likely want to generate operational metrics for monitoring (e.g. via StatsD or Prometheus Pushgateway).

To outline a few approaches:
  - **Option 1**: frequently run stream ingestion on a trigger, and then run this in the orchestration tool of choice like Airflow, Databricks Jobs, etc. e.g. 
    ```python
    (seven_day_avg
        .writeStream
        .outputMode("append") 
        .option("checkpointLocation", "/tmp/feast-workshop/q3/")
        .trigger(once=True)
        .foreachBatch(send_to_feast)
        .start())
    ```
  - **Option 2**: with Databricks, use Databricks Jobs to monitor streaming queries and auto-retry on a new cluster + on failure. See [Databricks docs](https://docs.databricks.com/structured-streaming/query-recovery.html#configure-structured-streaming-jobs-to-restart-streaming-queries-on-failure) for details.
  - **Option 3**: with Dataproc, configure [restartable jobs](https://cloud.google.com/dataproc/docs/concepts/jobs/restartable-jobs)
  - **Option 4** If you're using Flink, then consider configuring a [restart strategy](https://nightlies.apache.org/flink/flink-docs-release-1.15/docs/ops/state/task_failure_recovery/)

# Conclusion
By the end of this module, you will have learned how to build a full feature platform, with orchestrated batch transformations (using dbt + Airflow), orchestrated materialization (with Feast + Airflow), and pointers on orchestrating streaming transformations.

## Limitations
- Feast does not itself handle orchestration of transformation or materialization, and relies on the user to configure this with tools like dbt and Airflow. 
- Feast does not ensure consistency in transformation logic between batch and stream features

## Why Feast?
Feast abstracts away the need to think about data modeling in the online store and helps you:
- maintain fresh features in the online store by
  - ingesting batch features into the online store (via `feast materialize` or `feast materialize-incremental`)
  - ingesting streaming features into the online store (e.g. through `feature_store.push` or a Push server endpoint (`/push`))
- serve features (e.g. through `feature_store.get_online_features` or through feature servers)

# FAQ

### How do I iterate on features?
Once a feature view is in production, best practice is to create a new feature view (+ a separate dbt model) to generate new features or change existing features, so-as not to negatively impact prediction quality.

This means for each new set of features, you'll need to:
1. Define a new dbt model
2. Define a new Feast data source + feature view, and feature service (model version) that depends on these features.
3. Ensure the transformations + materialization jobs are executed at the right cadence with Airflow + dbt + Feast.

### How does this work in production?
Several things change:
- All credentials are secured as secrets
- dbt models are version controlled
- Production deployment of Airflow (e.g. syncing with a Git repository of DAGs, using k8s)
- Bundling dbt models with Airflow (e.g. via S3 like this [MWAA + dbt guide](https://docs.aws.amazon.com/mwaa/latest/userguide/samples-dbt.html))
- Airflow DAG parallelizes across feature views (instead of running a single `feature_store.materialize` across all feature views)
- Feast materialization is configured to be more scalable (e.g. using other Feast batch materialization engines [Bytewax](https://docs.feast.dev/reference/batch-materialization/bytewax), [Snowflake](https://docs.feast.dev/reference/batch-materialization/snowflake), [Lambda](https://docs.feast.dev/reference/batch-materialization/lambda), [Spark](https://docs.feast.dev/reference/batch-materialization/spark))