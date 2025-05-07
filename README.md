# ETL SCD1 Pipeline using Snowflake, AWS S3, and Python

## 📌 Project Overview

This project demonstrates how to build an **ETL pipeline implementing Slowly Changing Dimension Type 1 (SCD1)** using:

* **AWS S3** for file-based ingestion
* **Snowflake** for staging and data transformation
* **Snowflake Streams & Tasks** for real-time automation
* **Stored Procedures** for business logic
* **Python (Boto3)** for file uploads to S3

---

## 🧠 SCD1 Concept

**SCD1 (Slowly Changing Dimension Type 1)** overwrites old data with new data when a change is detected based on the defined business key (e.g., `CONTACTFIRSTNAME` + `CONTACTLASTNAME`).

---

## 📁 Architecture

```
Customer CSV files (Full + Change Data)
           ↓
       Python Script
(Upload to AWS S3 bucket)
           ↓
       Snowflake Stage
(Auto-ingested via External Stage)
           ↓
         RAW Table
    ↓                 ↓
 STREAM (Change Capture)   TASK (Scheduled Trigger)
           ↓
Stored Procedure (Merge logic)
           ↓
     Final SCD1 CUSTOMER Table
```

---

## 🛠️ Technologies Used

* Snowflake Cloud Data Warehouse
* AWS S3
* Python + Boto3
* Snowflake Streams, Tasks & Stored Procedures

---



---

## ✅ Step-by-Step Setup

### 1. Environment Setup

Install dependencies:

```bash
pip install boto3 pyyaml
```

Create `aws_credentials.yml`:

```yaml
aws_access_key_id: 'YOUR_ACCESS_KEY'
aws_secret_access_key: 'YOUR_SECRET_KEY'
aws_region_name: 'YOUR_REGION'
```

### 2. Upload Files to S3 (Python Notebook)

Notebook: `upload_to_s3.ipynb`

```python
import boto3, yaml

def upload_file_to_s3(file_name, bucket_name, folder_name):
    with open("aws_credentials.yml") as f:
        creds = yaml.safe_load(f)

    s3 = boto3.client('s3',
        aws_access_key_id=creds['aws_access_key_id'],
        aws_secret_access_key=creds['aws_secret_access_key'],
        region_name=creds['aws_region_name'])

    s3.upload_file(file_name, bucket_name, f"{folder_name}/{file_name}")

upload_file_to_s3("customer_full_data.csv", "your-bucket", "your-folder")
```

---

### 3. Snowflake SQL Scripts

#### Create Tables

```sql
CREATE OR REPLACE DATABASE SCD1_DB;
USE DATABASE SCD1_DB;

CREATE OR REPLACE TABLE CUSTOMER_RAW (
  CONTACTFIRSTNAME STRING,
  CONTACTLASTNAME STRING,
  PHONE STRING,
  CITY STRING
);

CREATE OR REPLACE TABLE CUSTOMER (
  CONTACTFIRSTNAME STRING,
  CONTACTLASTNAME STRING,
  PHONE STRING,
  CITY STRING
);
```

#### Create External Stage

```sql
CREATE OR REPLACE STAGE CUSTOMER_STAGE
URL = 's3://your-bucket/your-folder/'
STORAGE_INTEGRATION = your_integration;
```

#### Load Data from Stage

```sql
COPY INTO CUSTOMER_RAW
FROM @CUSTOMER_STAGE/customer_full_data.csv
FILE_FORMAT = (TYPE = 'CSV' FIELD_OPTIONALLY_ENCLOSED_BY='"' SKIP_HEADER=1);
```

#### Create Stream

```sql
CREATE OR REPLACE STREAM CUSTOMER_RAW_STREAM ON TABLE CUSTOMER_RAW;
```

#### Stored Procedure for SCD1

```sql
CREATE OR REPLACE PROCEDURE SP_MERGE_CUSTOMER()
RETURNS STRING
LANGUAGE SQL
AS
$$
MERGE INTO CUSTOMER T
USING (
    SELECT * FROM CUSTOMER_RAW_STREAM
) S
ON T.CONTACTFIRSTNAME = S.CONTACTFIRSTNAME
AND T.CONTACTLASTNAME = S.CONTACTLASTNAME
WHEN MATCHED THEN UPDATE SET
  T.PHONE = S.PHONE,
  T.CITY = S.CITY
WHEN NOT MATCHED THEN INSERT (
  CONTACTFIRSTNAME, CONTACTLASTNAME, PHONE, CITY
) VALUES (
  S.CONTACTFIRSTNAME, S.CONTACTLASTNAME, S.PHONE, S.CITY
);
$$;
```

#### Create & Start Task

```sql
CREATE OR REPLACE TASK MERGE_CUSTOMER_TASK
WAREHOUSE = COMPUTE_WH
SCHEDULE = '1 MINUTE'
AS
CALL SP_MERGE_CUSTOMER();

ALTER TASK MERGE_CUSTOMER_TASK RESUME;
```

---

## 📊 Testing

### 1. Upload `customer_full_data.csv`

```sql
SELECT COUNT(*) FROM CUSTOMER; -- Expect 80
```

### 2. Upload `customer_change_data.csv`

```sql
-- Validate changes
SELECT * FROM CUSTOMER WHERE CONTACTFIRSTNAME = 'Elizabeth' AND CONTACTLASTNAME = 'Yu';
SELECT * FROM CUSTOMER WHERE CONTACTFIRSTNAME = 'Kyung' AND CONTACTLASTNAME = 'Benitez';
SELECT * FROM CUSTOMER WHERE CONTACTFIRSTNAME = 'John' AND CONTACTLASTNAME = 'Wick';
```

---

## 📊 Output Sample

| FirstName | LastName | Phone        | City        |
| --------- | -------- | ------------ | ----------- |
| Elizabeth | Yu       | 626-555-3459 | Torrance    |
| Kyung     | Benitez  | 310-555-8855 | Pasadena    |
| John      | Wick     | 415-555-9999 | Los Angeles |

---

## 📈 Learning Outcomes

* Understand SCD1 in practice
* Hands-on with Snowflake CDC tools
* Python + S3 integration for ingestion
* Real-time data merge using Tasks + Streams

---

## 📣 Author

**\ Enoch Numo Tetteh**
Data Engineer
[LinkedIn] www.linkedin.com/in/enochnumotetteh

---


