# Spotify Data Pipeline on AWS

This project automates the extraction of data from the **Spotify API**, processes it with **AWS serverless** services, and prepares it for analysis in **Amazon Athena**.

---

## Architecture Summary

The workflow follows three main stages: **Extract**, **Transform**, and **Load (ETL)**. Each stage uses managed AWS services to schedule, process, store, catalog, and analyze the data end-to-end.
<img width="1536" height="1024" alt="Architecture Diagram" src="https://github.com/user-attachments/assets/a5c6db52-7b29-46ca-b96e-ff437f851a0f" />


---

## Detailed Pipeline Explanation

### Extract Stage

- **Spotify API & Python**  
  The pipeline begins with Python code retrieving music data (songs, playlists, stats, etc.) from the Spotify Web API.

- **Amazon CloudWatch**  
  Schedules **daily triggers** (EventBridge rules) to automate the data extraction process so it runs unattended.

- **AWS Lambda (Extraction)**  
  Executes the Python logic, pulls raw data from Spotify, and writes it to **Amazon S3** as raw JSON files.

---

### Transform Stage

- **Amazon S3 (Raw Data)**  
  Stores the extracted raw Spotify data securely and durably.

- **S3 Trigger (ObjectCreated)**  
  Monitors the raw-data prefix for new files. When a file arrives, it automatically triggers the transformation Lambda.

- **AWS Lambda (Transformation)**  
  Cleans, structures, and formats the raw Spotify data into analytics-ready files.  
  Typical steps include:
  - Removing duplicates  
  - Normalizing/formatting dates  
  - Standardizing fields and column names

- **Amazon S3 (Transformed Data)**  
  Saves the newly processed datasets (e.g., **artists**, **albums**, **songs**) for downstream analytics and querying.

---

### Load Stage

- **AWS Glue Crawler**  
  Scans the transformed data in S3 and infers its schema (column names, data types, partitions, etc.).

- **AWS Glue Data Catalog**  
  Stores the discovered metadata, making the datasets discoverable and queryable by AWS analytics services.

- **Amazon Athena**  
  Provides **serverless SQL analytics** over the transformed Spotify data. Users can run queries, generate reports, and visualize trends directly from S3.

---

## Technologies Used

- **Spotify API** – Data source  
- **Python** – Extraction & transformation code  
- **Amazon CloudWatch / EventBridge** – Automated scheduling  
- **AWS Lambda** – Serverless compute for extraction and transformation  
- **Amazon S3** – Scalable object storage for raw and transformed data  
- **AWS Glue (Crawler & Data Catalog)** – Schema inference and metadata management  
- **Amazon Athena** – Cloud-native SQL analytics

---

## Usage Instructions

1. **Set up AWS access** and IAM permissions for Lambda, S3, and Glue.  
2. **Connect Python code** to the Spotify API (client credentials or OAuth as needed).  
3. **Create S3 buckets/prefixes** for `raw_data/` and `transformed_data/`.  
4. **Configure CloudWatch (EventBridge)** for a daily schedule to trigger extraction.  
5. **Deploy Lambda functions** for both extraction and transformation.  
6. **Enable S3 notifications** (ObjectCreated) to trigger the transformation Lambda.  
7. **Configure a Glue Crawler** to catalog the transformed datasets.  
8. **Query with Athena** to analyze trends, build reports, and power dashboards.

---

This architecture delivers an **automated, serverless** ingestion and analytics pipeline for Spotify data—ideal for music analytics, reporting, and dashboarding.
