# Airflow DVC support

## Examples

The examples are provided in the `dags/` directory.

## Usage

The package provides the following core features:
* DVCUpdateOperator (for uploading data to DVC)
* DVCDownloadOperator (for downloading data from DVC)
* DVCUpdateSensor (for waiting for a file modification on DVC)

### DVCUpdateOperator (Uploading)

The upload operator supports various types of data inputs that you can feed into it.

**Uploading a string as a file:**
```python
    from airflow_dvc import DVCUpdateOperator, DVCStringUpload
    from datetime import datetime

    upload_task = DVCUpdateOperator(
        dvc_repo="<REPO_CLONE_URL>",
        files=[
            DVCStringUpload("data/1.txt", f"This will be saved into DVC. Current time: {datetime.now()}"),
        ],
        task_id='update_dvc',
    )
```

**Uploading local file using its path:**
```python
    from airflow_dvc import DVCUpdateOperator, DVCPathUpload

    upload_task = DVCUpdateOperator(
        dvc_repo="<REPO_CLONE_URL>",
        files=[
            DVCPathUpload("data/1.txt", "~/local_file_path.txt"),
        ],
        task_id='update_dvc',
    )
```

**Upload content generated by a python function:**
```python
    from airflow_dvc import DVCUpdateOperator, DVCCallbackUpload

    upload_task = DVCUpdateOperator(
        dvc_repo="<REPO_CLONE_URL>",
        files=[
            DVCCallbackUpload("data/1.txt", lambda: "Test data"),
        ],
        task_id='update_dvc',
    )
```

**Uploading file from S3:**
This is specially useful when you have a workflow that uses [S3Hook](https://airflow.apache.org/docs/apache-airflow/1.10.14/_modules/airflow/hooks/S3_hook.html) to temporarily save the data between tasks.

```python
from airflow import DAG
from airflow.operators.python_operator import PythonOperator
from airflow.providers.amazon.aws.hooks.s3 import S3Hook
from datetime import datetime, timedelta

from io import StringIO
import pandas as pd
import requests

from airflow_dvc import DVCUpdateOperator, DVCS3Upload

s3_conn_id = 's3-conn'
bucket = 'astro-workshop-bucket'
state = 'wa'
date = '{{ yesterday_ds_nodash }}'

def upload_to_s3(state, date):
    '''Grabs data from Covid endpoint and saves to flat file on S3
    '''
    # Connect to S3
    s3_hook = S3Hook(aws_conn_id=s3_conn_id)

    # Get data from API
    url = 'https://covidtracking.com/api/v1/states/'
    res = requests.get(url+'{0}/{1}.csv'.format(state, date))

    # Save data to CSV on S3
    s3_hook.load_string(res.text, '{0}_{1}.csv'.format(state, date), bucket_name=bucket, replace=True)

def process_data(state, date):
    '''Reads data from S3, processes, and saves to new S3 file
    '''
    # Connect to S3
    s3_hook = S3Hook(aws_conn_id=s3_conn_id)

    # Read data
    data = StringIO(s3_hook.read_key(key='{0}_{1}.csv'.format(state, date), bucket_name=bucket))
    df = pd.read_csv(data, sep=',')

    # Process data
    processed_data = df[['date', 'state', 'positive', 'negative']]

    # Save processed data to CSV on S3
    s3_hook.load_string(processed_data.to_string(), 'dvc_upload.csv', bucket_name=bucket, replace=True)

# Default settings applied to all tasks
default_args = {
    'owner': 'airflow',
    'depends_on_past': False,
    'email_on_failure': False,
    'email_on_retry': False,
    'retries': 1,
    'retry_delay': timedelta(minutes=1)
}

with DAG('intermediary_data_storage_dag',
         start_date=datetime(2021, 1, 1),
         max_active_runs=1,
         schedule_interval='@daily',
         default_args=default_args,
         catchup=False
         ) as dag:

    generate_file = PythonOperator(
        task_id='generate_file_{0}'.format(state),
        python_callable=upload_to_s3,
        op_kwargs={'state': state, 'date': date}
    )

    process_data = PythonOperator(
        task_id='process_data_{0}'.format(state),
        python_callable=process_data,
        op_kwargs={'state': state, 'date': date}
    )
    
    upload_to_dvc = DVCUpdateOperator(
        dvc_repo="<REPO_CLONE_URL>",
        files=[
            DVCS3Upload("dvc_path/data.txt", s3_conn_id, bucket, 'dvc_upload.csv'),
        ],
        task_id='update_dvc',
    )

    generate_file >> process_data
    process_data >> upload_to_dvc
```

**Uploading file from S3, but using task arguments:**
Instead of passing list as a files parameter you can pass function as would do in case of PythonOperator:
```python
from airflow import DAG
from airflow.operators.python_operator import PythonOperator
from airflow.providers.amazon.aws.hooks.s3 import S3Hook
from datetime import datetime, timedelta

from io import StringIO
import pandas as pd
import requests

from airflow_dvc import DVCUpdateOperator, DVCS3Upload

s3_conn_id = 's3-conn'
bucket = 'astro-workshop-bucket'
state = 'wa'
date = '{{ yesterday_ds_nodash }}'

def upload_to_s3(state, date):
    '''Grabs data from Covid endpoint and saves to flat file on S3
    '''
    # Connect to S3
    s3_hook = S3Hook(aws_conn_id=s3_conn_id)

    # Get data from API
    url = 'https://covidtracking.com/api/v1/states/'
    res = requests.get(url+'{0}/{1}.csv'.format(state, date))

    # Save data to CSV on S3
    s3_hook.load_string(res.text, '{0}_{1}.csv'.format(state, date), bucket_name=bucket, replace=True)

def process_data(state, date):
    '''Reads data from S3, processes, and saves to new S3 file
    '''
    # Connect to S3
    s3_hook = S3Hook(aws_conn_id=s3_conn_id)

    # Read data
    data = StringIO(s3_hook.read_key(key='{0}_{1}.csv'.format(state, date), bucket_name=bucket))
    df = pd.read_csv(data, sep=',')

    # Process data
    processed_data = df[['date', 'state', 'positive', 'negative']]

    # Save processed data to CSV on S3
    s3_hook.load_string(processed_data.to_string(), '{0}_{1}_processed.csv'.format(state, date), bucket_name=bucket, replace=True)

def get_files_for_upload(state, date):
    return [
        DVCS3Upload("dvc_path/data.txt", s3_conn_id, bucket, '{0}_{1}_processed.csv'.format(state, date)),
    ]
    
# Default settings applied to all tasks
default_args = {
    'owner': 'airflow',
    'depends_on_past': False,
    'email_on_failure': False,
    'email_on_retry': False,
    'retries': 1,
    'retry_delay': timedelta(minutes=1)
}

with DAG('intermediary_data_storage_dag',
         start_date=datetime(2021, 1, 1),
         max_active_runs=1,
         schedule_interval='@daily',
         default_args=default_args,
         catchup=False
         ) as dag:

    generate_file = PythonOperator(
        task_id='generate_file_{0}'.format(state),
        python_callable=upload_to_s3,
        op_kwargs={'state': state, 'date': date}
    )

    process_data = PythonOperator(
        task_id='process_data_{0}'.format(state),
        python_callable=process_data,
        op_kwargs={'state': state, 'date': date}
    )
    
    # Passing a function as files paramterer (it sould return a list of DVCUpload objects)
    # Also we specify op_kwargs to allow passing of parameters as in case of normal PythonOperator
    upload_to_dvc = DVCUpdateOperator(
        dvc_repo="<REPO_CLONE_URL>",
        files=get_files_for_upload,
        task_id='update_dvc',
        op_kwargs={'state': state, 'date': date}
    )

    generate_file >> process_data
    process_data >> upload_to_dvc
```

### DVCDownloadOperator (Downloading)

We can use `VCDownloadOperator` similarily to the `DVCUpdateOperator`. The syntax is the same:
```python
    from airflow_dvc import DVCDownloadOperator, DVCCallbackDownload

    # Download DVC file data/1.txt and print it on the screen
    upload_task = DVCDownloadOperator(
        dvc_repo="<REPO_CLONE_URL>",
        files=[
            DVCCallbackDownload("data/1.txt", lambda content: print(content)),
        ],
        task_id='update_dvc',
    )
```

The `DVCDownload` implementations are similar to `DVCUpload`.

### DVCSensor

`DVCSensor` will allow you to pause the DAG run until the specified file will be updated.
The sensor checks the date of the latest DAG run and compares it with timestamp of meta DVC file in the repo.

```python
from datetime import datetime
from airflow import DAG
from airflow.operators.dummy_operator import DummyOperator
from airflow.operators.bash_operator import BashOperator

from airflow_dvc import DVCUpdateSensor


with DAG('dvc_sensor_example', description='Another tutorial DAG',
    start_date=datetime(2017, 3, 20),
    catchup=False,
) as dag:

    dummy_task = DummyOperator(task_id='dummy_task', dag=dag)

    sensor_task = DVCUpdateSensor(
        task_id='dvc_sensor_task',
        dag=dag,
        dvc_repo="<REPO_CLONE_URL>",
        files=["data/1.txt"],
    )

    task = BashOperator(
        task_id='task_triggered_by_sensor',
        bash_command='echo "OK" && ( echo $[ ( $RANDOM % 30 )  + 1 ] > meowu.txt ) && cat meowu.txt')

    dummy_task >> sensor_task >> task

```