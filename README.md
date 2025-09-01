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


## ⚙️ Configuration

Set these **environment variables** on the Lambdas:

### `data_extraction` (Lambda)
- `CLIENT_ID` – Spotify App Client ID
- `CLIENT_SECRET` – Spotify App Client Secret (only for client-credentials)
- `REDIRECT_URI` – if using user auth to access playlists
- `SPOTIFY_SECRET_ID` – Secrets Manager ID that stores `{ client_id, client_secret, refresh_token, redirect_uri }` (optional; used if you need user tokens)
- `BUCKET_NAME` – S3 bucket (e.g., `spotify-etl-pipeline-hp`)
- `RAW_PREFIX` – `raw_data/to_be_processed/`

### `data_transformation` (Lambda)
- `BUCKET_NAME` – same S3 bucket
- `RAW_PREFIX` – `raw_data/to_be_processed/`
- `PROCESSED_PREFIX` – `raw_data/processed_data/`
- `TGT_PREFIX` – `transformed_data/` (subfolders: `songs_data/`, `album_data/`, `artists_data/`)

> **IAM:** Both Lambdas need `s3:GetObject`, `s3:PutObject`, `s3:ListBucket` on the bucket/prefixes. Extraction also needs `secretsmanager:GetSecretValue` if using refresh tokens, and optional `logs:*` for CloudWatch Logs.

Minimal S3 policy snippet:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {"Effect":"Allow","Action":["logs:CreateLogGroup","logs:CreateLogStream","logs:PutLogEvents"],"Resource":"*"},
    {"Effect":"Allow","Action":["s3:GetObject","s3:PutObject"],"Resource":[
      "arn:aws:s3:::YOUR_BUCKET/raw_data/*",
      "arn:aws:s3:::YOUR_BUCKET/transformed_data/*"
    ]},
    {"Effect":"Allow","Action":["s3:ListBucket"],"Resource":"arn:aws:s3:::YOUR_BUCKET"},
    {"Effect":"Allow","Action":["secretsmanager:GetSecretValue"],"Resource":"arn:aws:secretsmanager:*:*:secret:*"}
  ]
}


