# Healthcare Data Pipeline using AWS Glue, S3, Lambda, and SNS
--------------------------------------------------------------

## Overview
This project implements a **serverless data pipeline** using AWS services like **S3, Glue, Lambda, SNS, and SQS** to process healthcare-related data efficiently. The pipeline ingests raw medical and patient data, performs transformations, and stores the processed data for further analysis using **AWS Glue and Athena**.

## Architecture
The data pipeline consists of the following components:

1. **Amazon S3**: Stores raw and processed data.
2. **S3 Event Notification**: Triggers an event when new files are uploaded.
3. **Amazon SNS / SQS**: Used to manage file upload events and trigger processing.
4. **AWS Lambda**: Automatically triggers AWS Glue when new files are uploaded.
5. **AWS Glue**:
   - Reads CSV files from S3.
   - Cleans and transforms the data.
   - Stores processed data in S3 (Parquet format).
6. **AWS Glue Data Catalog**: Maintains metadata for querying via Athena.
7. **Amazon Athena / Redshift**: Used for querying and analytics.
8. **AWS SNS**: Sends job success/failure notifications.

## Workflow
1. **File Upload**: CSV files are uploaded to an S3 bucket (`Health_project/`).
2. **S3 Event Notification**: Sends an event to SNS/SQS.
3. **AWS Lambda**: Reads the event and triggers a Glue ETL job.
4. **AWS Glue Job**:
   - Reads new files from S3.
   - Cleans and processes the data (handling missing values, deduplication, aggregations, etc.).
   - Joins patient and medical data.
   - Writes processed data to S3 (`Output_data/delta/`) in Parquet format.
5. **Glue Data Catalog**: Updates metadata for Athena.
6. **Amazon SNS**: Sends notifications on job **start, success, or failure**.
7. **Athena / Redshift**: Used to query the transformed data.

## Technologies Used
- **AWS S3** - Storage for raw & processed data.
- **AWS Glue** - Serverless ETL for data transformation.
- **AWS Lambda** - Serverless function to trigger Glue jobs.
- **AWS SQS/SNS** - Manages file upload events.
- **AWS Athena / Redshift** - Querying processed data.
- **Apache Spark (PySpark)** - Data transformation in Glue.

## Code Explanation
### **1. Lambda Function** (Trigger Glue Job)
- Listens to SQS messages (file upload events).
- Starts the Glue job when two files arrive within a time window.
- Example snippet:
```python
import json
import boto3
from datetime import datetime, timedelta

glue_client = boto3.client('glue')
SQS_QUEUE_URL = "https://sqs.us-west-2.amazonaws.com/XXXXXX/New"

uploaded_files = []
upload_time_window = timedelta(minutes=5)

def lambda_handler(event, context):
    global uploaded_files
    for record in event['Records']:
        s3_message = json.loads(record['body'])
        bucket_name = s3_message['Records'][0]['s3']['bucket']['name']
        file_key = s3_message['Records'][0]['s3']['object']['key']
        uploaded_files.append({'bucket_name': bucket_name, 'file_key': file_key, 'timestamp': datetime.now()})
        if len(uploaded_files) == 2:
            start_glue_job()
            uploaded_files.clear()

def start_glue_job():
    response = glue_client.start_job_run(JobName='Project_Health')
    print(f"Glue job started with run ID: {response['JobRunId']}")
```

### **2. AWS Glue Job** (ETL Process)
- Reads CSV files from S3.
- Cleans and joins patient & medical data.
- Writes transformed data to S3 in Parquet format.
- Example snippet:
```python
from pyspark.sql import functions as F
from awsglue.context import GlueContext
from awsglue.dynamicframe import DynamicFrame

glueContext = GlueContext(SparkContext.getOrCreate())

df_patients = glueContext.create_dynamic_frame.from_options(
    connection_type="s3", connection_options={"paths": ["s3://bucket/Health_project/patients.csv"]}, format="csv"
).toDF()

df_medical = glueContext.create_dynamic_frame.from_options(
    connection_type="s3", connection_options={"paths": ["s3://bucket/Health_project/medical.csv"]}, format="csv"
).toDF()

# Data Cleaning and Transformation
joined_df = df_patients.join(df_medical, "patient_id", "inner")
abnormal_tests = df_medical.withColumn("is_abnormal", F.when(F.col("test_results") != "Normal", 1).otherwise(0))

target_path = "s3://bucket/Output_data/delta/"
joined_df.write.format("parquet").mode("overwrite").save(target_path)
```

## Deployment Steps
### **1. Set up S3 bucket**
- Create an S3 bucket (`awsashik123bucket`).
- Upload sample CSV files under `Health_project/`.

### **2. Create SQS Queue & SNS Topic**
- Create an **SQS queue** to collect S3 events.
- Create an **SNS topic** for Glue job status notifications.

### **3. Configure S3 Event Notifications**
- Add an **event notification** to the bucket to send new file events to SQS.

### **4. Deploy Lambda Function**
- Create a Lambda function to trigger AWS Glue jobs.
- Attach the correct IAM permissions.

### **5. Create AWS Glue Job**
- Define a Glue job that processes the data.
- Configure the job to read from S3, transform the data, and write output to S3.

### **6. Test the Pipeline**
- Upload test files to S3.
- Verify that Glue processes the data and SNS sends notifications.
- Query the data using Athena.

## Monitoring & Logging
- **AWS CloudWatch**: Monitor Lambda & Glue logs.
- **SNS Notifications**: Receive alerts for job success/failure.
- **Athena Queries**: Validate processed data.

## Conclusion
This **serverless ETL pipeline** efficiently processes healthcare data using AWS Glue, Lambda, and SNS, ensuring real-time processing with minimal infrastructure management.

---

### Future Enhancements
- Implement AWS Step Functions for better orchestration.
- Enable Glue job bookmarking for incremental data processing.
- Use AWS Lake Formation for data governance.
- Integrate with QuickSight for visualization.

---
🚀 **Contributors & Maintainers**: Feel free to raise issues or contribute! 🛠

