# Spotify Global Charts – Serverless ETL on AWS

Automated ETL that pulls top songs from the Spotify Web API, stores raw JSON in S3, transforms to tabular data (artists, albums, songs), catalogs the schema with AWS Glue, and queries with Amazon Athena. Scheduled to run daily so you can roll up weekly insights (top tracks/artists/albums) across the year.

## 🏗️ Architecture

- **Trigger:** Amazon CloudWatch (EventBridge) daily cron
- **Extract:** Lambda (`data_extraction`) calls the Spotify API  
  - Optional: reads a **refresh token** from Secrets Manager if user auth is needed
- **Raw zone:** S3 `raw_data/to_be_processed/`
- **Transform:** S3 Put event → Lambda (`data_transformation`) → CSV/Parquet into `transformed_data/`
- **Catalog:** AWS Glue Crawler updates Glue Data Catalog
- **Query:** Amazon Athena for analytics / dashboards

See `docs/architecture.mmd` (Mermaid) in this repo for the diagram.

## 🗂️ Repository layout

