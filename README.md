# Data-Pipeline-Web-GCS-Postgres_database
![Add a heading (1)](https://github.com/krissemmy/Data-Pipeline-Web-GCS-Postgres_database/assets/119800888/46ba7f0c-9afc-4225-8d3e-2accb4db69bd)


## Overview
• We have an Extract, Transform, and Load (ETL) pipeline designed to retrieve New York City Green taxi data from the DataTalks GitHub Repository. This data is then loaded into a Google Cloud Storage (GCS) Bucket before being transferred to a specific table within a PostgreSQL database.

• The pipeline is scheduled to run on a monthly basis, fetching data for the respective month.

• We've constructed this pipeline using the GCS Hook in combination with a custom PostgreSQL connection.

• To enhance scalability, we've encapsulated the data pipeline within a Docker container and execute it using the Celery executor. This approach allows us to easily scale up our operations as needed.

## Setup (official)

### Requirements
1. Upgrade docker-compose version:2.x.x+
2. Allocate memory to docker between 4gb-8gb
3. Python: version 3.8+


### Set Airflow

1.  On Linux, the quick-start needs to know your host user-id and needs to have group id set to 0.
    Otherwise the files created in `dags`, `logs` and `plugins` will be created with root user.

2.  set your airflow user id using:

    ```bash
    echo -e "AIRFLOW_UID=$(id -u)" > .env
    ```

    For Windows same as above.

    Create `.env` file with the content below as:

    ```
    AIRFLOW_UID=50000
    ```
3. Download or import the docker setup file from airflow's website : Run this on terminal
```
curl -LfO 'https://airflow.apache.org/docs/apache-airflow/stable/docker-compose.yaml'
```
4. Create "Dockerfile" use to build airflow container image.

5. Add this to the Dockerfile:
```
FROM apache/airflow:2.6.3
# For local file running
RUN pip install --no-cache-dir "apache-airflow==${AIRFLOW_VERSION}" pandas sqlalchemy psycopg2-binary
USER root
RUN apt-get update \
&& apt-get install -y --no-install-recommends \
vim \
&& apt-get autoremove -yqq --purge \
&& apt-get clean \
&& rm -rf /var/lib/apt/lists/*
WORKDIR $AIRFLOW_HOME
USER $AIRFLOW_UID
```
6. Go into the docker-compose.yaml file for the airflow and replace the build context with:
```
 build:
    context: .
    dockerfile: ./Dockerfile
```
7. Save all the modified files

8. Build image: docker-compose build

9. Initialize airflow db; docker-compose up airflow-init

10. Initialize all the other services: docker-compose up

11. Connect external postgres container to the airflow container by assigning the airflow_default network to the postgres container: see the yaml file.
run below command to see the name of your airflow network
```bash
docker network ls
```

12. Check for postgres db access from the airflow container.

### SetUp GCP for Local System (Local Environment Oauth-authentication)
1. Establish a Google Cloud Platform (GCP) PROJECT.
2. Generate a service account with permissions: Include roles such as Editor, Storage Admin, Storage Object Admin, and BigQuery Admin.
3. Generate credential keys for the service account and save them for future use.
4. Modify the name and location as required.
```bash
cd ~ && mkdir -p ~/.google/credentials/

mv <path/to/your/service-account-authkeys>.json ~/.google/credentials/google_credentials.json
```

   Below is an example
   
```
mv  /home/krissemmy/Downloads/alt-data-engr-1dfdbf9f8dbf.json ~/.google/credentials/google_credentials.json
```
5. Install gcloud on system : open new terminal and run    (follow this link to install gcloud-sdk : https://cloud.google.com/sdk/docs/install-sdk)

    ```bash
    gcloud -v
    ```
  to see if its installed successfully
6. Set the google applications credentials environment variable

  ```bash
  export GOOGLE_APPLICATION_CREDENTIALS="/path/to/.json-file"
  ```

  Below is an example

  ```bash
  export GOOGLE_APPLICATION_CREDENTIALS = "/home/krissemmy/.google/credentials/google_credentials.json"
  ```
7. Run gcloud auth application-default login
8. Redirect to the website and authenticate local environment with the cloud environment

## Enable API
perform the following on your Google Cloud Platform
1. Enable Identity  and Access management API
2. Enable IAM Service Account Credentials API


## Update docker-compose file and Dockerfile
1. Add google credentials "GOOGLE_APPLICATION_CREDENTIALS" and project_id and bucket name
    ```
        GOOGLE_APPLICATION_CREDENTIALS: /.google/credentials/google_credentials.json
        AIRFLOW_CONN_GOOGLE_CLOUD_DEFAULT: 'google-cloud-platform://?extra__google_cloud_platform__key_path=/.google/credentials/google_credentials.json'

        GCP_PROJECT_ID: "my-project-id"
        GCP_GCS_BUCKET: "my-bucket"
    ```
2. Add the below line to the volumes of the airflow documentation

    ```
    ~/.google/credentials/:/.google/credentials:ro
    ```
    ![image](https://github.com/krissemmy/Data-Pipeline-Web-GCS-BQ/assets/119800888/bc2396d1-b9d5-4d6d-806c-ccc9253d9c89)


3. build airflow container image with:
```bash
docker-compose build
```
4. Initialize airflow db;
```bash
docker-compose up airflow-init
```
5. Initialize all the other services: 
```bash
docker-compose up
```
6. Inside plugins/web/operators folder is the python file with the WebToGCSOperator.
7. Inside dags folder is web_gcs_bq.py file with all the neccessary dag code, you can make modifications to the time schedules and any other thing you feel like
7. To check if all containers are running fine and healthy, open a new terminal run the below
```bash
docker ps
```
8. You can connect to your Airflow webserver interface at http://localhost:8080/
9. Default username and password is 

username : airflow

password : airflow



# Possible Improvement: Efficient Data Loading to PostgreSQL
### Current Approach and Limitations
Our current data loading process within the Apache Airflow DAG is dependent on Pandas and SQLAlchemy, both of which are powerful tools for data manipulation and database interactions. Nevertheless, there are certain drawbacks associated with this approach:

Performance: When dealing with substantial datasets, utilizing Pandas and SQLAlchemy can result in sluggish data loading processes. This is especially noticeable when dealing with millions of records, as it can lead to excessive memory consumption and prolonged processing durations.

Flexibility: The existing approach may lack the flexibility required for certain scenarios. Particularly, when utilizing the copy_expert method, it may not automatically identify the data schema, necessitating additional manual configuration steps.

Jinja Templating: There are instances where employing Jinja templating for generating dynamic file names in conjunction with Pandas may not yield the expected results. This can pose challenges for automation and dynamic file handling.

### Proposed Solution
To overcome these limitations and enhance the data loading process, we recommend harnessing the capabilities of Apache Airflow's PostgreSQL Hook and Operator. Here's how they can be advantageous:

Enhanced Performance: The PostgreSQL Hook and Operator provide efficient methods for directly loading data into PostgreSQL. This can lead to substantial performance improvements, especially when handling large datasets. The copy_expert method, in particular, is optimized for high-speed data loading.

Automated Schema Detection: The PostgreSQL Hook and Operator can automatically identify the schema of the data being loaded, reducing the necessity for manual setup. This simplifies and streamlines the data loading procedure.

Jinja Templating Compatibility: With the PostgreSQL Operator, you can seamlessly incorporate Jinja templating for dynamic file names and other parameters. This ensures greater flexibility and automation within your workflow.

### Implementation Considerations
When integrating the PostgreSQL Hook and Operator into your data loading process, keep the following steps in mind:

Incorporate the PostgresHook and PostgresOperator classes into your DAG for seamless operation.

Harness the power of the copy_expert method for streamlined data loading, particularly beneficial when dealing with extensive datasets.

Employ Jinja templating within the PostgresOperator to dynamically configure file names and other parameters, enhancing flexibility.

By embracing this enhanced approach, we can elevate the efficiency and adaptability of our data loading process, guaranteeing smoother and more expedient data ingestion into PostgreSQL.
