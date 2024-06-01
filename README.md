
# Data Pipeline with Airflow, Kafka, Spark, and Cassandra

This project sets up a data pipeline that fetches data from an API, streams it to Kafka, processes it with Spark, and stores the results in Cassandra. Below are the detailed steps to set up and run the pipeline.

## Prerequisites

Ensure you have Docker installed on your machine.

## Steps

### 1. Clone the Repository

Clone this repository to your local machine:
```bash
git clone https://github.com/MortezaMahdavi/airflow_kafka_spark_streaming.git
cd airflow_kafka_spark_streaming
```

### 2. Create the Docker Network

```bash
docker network create airflow-kafka
```

### 3. Pull Docker Images and Create Containers

Navigate to the directory where your `docker-compose-airflow.yml` and `docker-compose-kafka.yml` files are located.

Start Airflow and PostgreSQL services:
```bash
docker-compose -f docker-compose-airflow.yml up -d
```

Start Kafka, Zookeeper, Spark, and Cassandra services:
```bash
docker-compose -f docker-compose-kafka.yml up -d
```

### 4. Install Dependencies in Airflow Docker Container

Now you have a running Airflow container and you can access the UI at [http://localhost:8080](http://localhost:8080). We can go into the Airflow container and install the libraries and packages.

First, get the container ID or name for the Airflow webserver:
```bash
docker ps
```

Exec into the Airflow container:
```bash
docker exec -it <airflow_webserver_container_id_or_name> bash
```

Install dependencies:
```bash
pip install -r /usr/local/airflow/requirements.txt
```

Exit the container:
```bash
exit
```

### 5. Create Kafka Topic Using Kafka UI

1. Open Kafka UI in your web browser by navigating to `http://localhost:8888`.
2. Click on your Kafka cluster (e.g., `local`).
3. Go to the "Topics" tab.
4. Click "Create Topic".
5. Enter the topic name: `random_names`.
6. Set the number of partitions to 3.
7. Click "Create".

### 6. Create Table in Cassandra

Exec into the Cassandra container:
```bash
docker exec -it cassandra /bin/bash
```

Access the `cqlsh` CLI with the necessary username and password variables:
```bash
cqlsh -u cassandra -p cassandra
```

Create the keyspace and table:
```cql
CREATE KEYSPACE spark_streaming WITH replication = {'class':'SimpleStrategy','replication_factor':1};
CREATE TABLE spark_streaming.random_names (
    full_name text PRIMARY KEY,
    gender text,
    location text,
    city text,
    country text,
    postcode int,
    latitude float,
    longitude float,
    email text
);
DESCRIBE spark_streaming.random_names;
```

Exit `cqlsh` and the container:
```bash
exit
exit
```

### 7. Add JAR Files to Spark and Submit Job

Copy JAR files from the repository's `jars` folder to the Spark container's `jars` folder:
```bash
docker cp ./jars spark_master:/opt/bitnami/spark/jars
```

Submit the Spark job:
```bash
docker exec -it spark_master spark-submit --master local[2] --packages org.apache.spark:spark-sql-kafka-0-10_2.12:3.2.1,com.datastax.spark:spark-cassandra-connector_2.12:3.0.0 /home/spark_streaming.py
```

This command runs the `spark_streaming.py` script using Spark's local mode with 2 cores. It includes the necessary packages for Kafka and Cassandra integration:
- `org.apache.spark:spark-sql-kafka-0-10_2.12:3.2.1` for Kafka.
- `com.datastax.spark:spark-cassandra-connector_2.12:3.0.0` for Cassandra.

### 8. Explanation of Python Files and Workflow

#### `stream_to_kafka.py`

This script performs the following tasks:
1. **Fetches data from a public API** (`https://randomuser.me/api/?results=1`) every 10 seconds.
2. **Formats the data** to include necessary fields.
3. **Sends the formatted data to a Kafka topic (`random_names`)** using the `kafka-python` library.

#### Airflow DAG (`stream_to_kafka_dag.py`)

This Airflow DAG:
1. **Schedules the `stream_to_kafka.py` script** to run at a specified interval (daily, in this example).

```python
from datetime import timedelta
from airflow import DAG
from airflow.operators.python_operator import PythonOperator
from datetime import datetime
from stream_to_kafka import start_streaming

start_date = datetime(2018, 12, 21, 12, 12)

default_args = {
    'owner': 'airflow',
    'start_date': start_date,
    'retries': 1,
    'retry_delay': timedelta(seconds=5)
}

with DAG('random_people_names', default_args=default_args, schedule_interval='0 1 * * *', catchup=False) as dag:
    data_stream_task = PythonOperator(
        task_id='kafka_data_stream',
        python_callable=start_streaming,
        dag=dag,
    )

    data_stream_task
```

#### `spark_streaming.py`

This Spark Structured Streaming job:
1. **Reads streaming data from the Kafka topic (`random_names`)**.
2. **Processes the data** (e.g., extracts fields from JSON messages).
3. **Saves the processed data to a Cassandra table** (`spark_streaming.random_names`).

```python
import logging
from pyspark.sql import SparkSession
from pyspark.sql.types import StructType, StructField, FloatType, IntegerType, StringType
from pyspark.sql.functions import from_json, col

logging.basicConfig(level=logging.INFO,
                    format='%(asctime)s:%(funcName)s:%(levelname)s:%(message)s')
logger = logging.getLogger("spark_structured_streaming")

def create_spark_session():
    try:
        spark = SparkSession \
            .builder \
            .appName("SparkStructuredStreaming") \
            .config("spark.cassandra.connection.host", "cassandra") \
            .config("spark.cassandra.connection.port", "9042") \
            .config("spark.cassandra.auth.username", "cassandra") \
            .config("spark.cassandra.auth.password", "cassandra") \
            .getOrCreate()
        spark.sparkContext.setLogLevel("ERROR")
        logging.info('Spark session created successfully')
    except Exception as e:
        logging.error(f"Couldn't create the spark session: {e}")
    return spark

def create_initial_dataframe(spark_session):
    try:
        df = spark_session \
            .readStream \
            .format("kafka") \
            .option("kafka.bootstrap.servers", "kafka1:19092,kafka2:19093,kafka3:19094") \
            .option("subscribe", "random_names") \
            .option("failOnDataLoss", "false") \
            .option("delimiter", ",") \
            .option("startingOffsets", "earliest") \
            .load()
        logging.info("Initial dataframe created successfully")
    except Exception as e:
        logging.warning(f"Initial dataframe couldn't be created due to exception: {e}")
    return df

def create_final_dataframe(df, spark_session):
    schema = StructType([
        StructField("full_name", StringType(), False),
        StructField("gender", StringType(), False),
        StructField("location", StringType(), False),
        StructField("city", StringType(), False),
        StructField("country", StringType(), False),
        StructField("postcode", IntegerType(), False),
        StructField("latitude", FloatType(), False),
        StructField("longitude", FloatType(), False),
        StructField("email", StringType(), False)
    ])

    df = df.selectExpr("CAST(value AS STRING)").select(from_json(col("value"), schema).alias("data")).select("data.*")
    return df

def start_streaming(df):
    logging.info("Streaming is being started...")
    my_query = (df.writeStream
                  .format("org.apache.spark.sql.cassandra")
                  .outputMode("append")
                  .option("checkpointLocation", "/opt/bitnami/spark/data/checkpoints/random_names")
                  .options(table="random_names", keyspace="spark_streaming")
                  .start())
    return my_query.awaitTermination()

def write_streaming_data():
    spark = create_spark_session()
    df = create_initial_dataframe(spark)
    df_final = create_final_dataframe(df, spark)
    start_streaming(df_final)

if __name__ == '__main__':
    write_streaming_data()
```

### Workflow Explanation

1. **Network Creation**: Establishes a Docker network for communication between containers.
2. **Service Setup**: Docker Compose files set up Airflow, Kafka, Spark, and Cassandra containers.
3. **Dependencies Installation**: Airflow dependencies are installed in the Airflow container to ensure DAGs can run.
4. **Kafka Topic Creation**: Sets up a Kafka topic to which data will be streamed via Kafka UI.
5. **Cassandra Table Creation**: Sets up a table in Cassandra where processed data from Spark will be stored.
6. **Spark Job Submission**: Copies necessary JAR files to Spark and submits the Spark job to start processing the streamed data using the specified packages.

By following these steps, we establish a data pipeline that:
-

 Fetches data from an API,
- Streams it to Kafka,
- Processes it with Spark,
- Stores the results in Cassandra.