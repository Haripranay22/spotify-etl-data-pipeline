# Spotify Global Charts – Serverless ETL on AWS

Automated ETL that pulls top songs from the Spotify Web API, stores raw JSON in S3, transforms to tabular data (artists, albums, songs), catalogs the schema with AWS Glue, and queries with Amazon Athena. Scheduled to run daily so you can roll up weekly insights (top tracks/artists/albums) across the year.

## ✨ Features

- Daily extraction from Spotify (search/playlists/tracks depending on your config)
- Durable raw zone in S3; idempotent transform with file move to `processed_data/`
- Clean normalized tables: `artists`, `albums`, `songs`
- Handles Spotify’s mixed album release date granularities (`year|month|day`)
- Glue Crawler + Data Catalog for Athena SQL analytics
- Packaged for AWS Lambda with a single dependency Layer (Spotipy + Pandas)

---

## 🧠 How it works (at a glance)

1. **Extract**  
   EventBridge (CloudWatch) daily cron → **Lambda `data_extraction`** → Spotify API → write JSON to  
   `s3://<BUCKET>/raw_data/to_be_processed/`.

2. **Transform**  
   S3 ObjectCreated → **Lambda `data_transformation`** → parse/normalize → write CSV/Parquet to  
   `s3://<BUCKET>/transformed_data/{artists_data|album_data|songs_data}/` and move raw JSON to  
   `s3://<BUCKET>/raw_data/processed_data/`.

3. **Load / Query**  
   **Glue Crawler** infers schema → **Glue Data Catalog** → **Athena** queries.

> You can aggregate daily results into weekly views in Athena (examples below).

---

## 🗂️ Suggested repository layout

.
├─ lambda/
│ ├─ extraction/
│ │ └─ lambda_function.py
│ ├─ transform/
│ │ └─ lambda_function.py
│ └─ layers/ # build scripts/zips for Spotipy & Pandas
├─ sql/
│ ├─ athena_example_queries.sql
│ └─ athena_create_views.sql
└─ README.md

yaml
Copy code

---

## ✅ Prerequisites

- AWS account with access to **S3**, **Lambda**, **EventBridge**, **Glue**, **Athena**
- Python **3.11** Lambda runtime
- Spotify Developer app (Client ID/Secret)
- (Optional) AWS **Secrets Manager** entry with a **refresh token** if you need user-level endpoints  
  (some playlist endpoints require a user token rather than client-credentials)

---

## 🔑 Spotify setup

1. Create an app at <https://developer.spotify.com/dashboard>.  
2. Add this Redirect URI in the app **Edit Settings** (for local token generation):  
   `http://127.0.0.1:8888/callback`
3. (Optional, only if you need user endpoints) obtain a **refresh token** locally:

   ```python
   import spotipy
   from spotipy.oauth2 import SpotifyOAuth

   sp = spotipy.Spotify(auth_manager=SpotifyOAuth(
       client_id="YOUR_CLIENT_ID",
       client_secret="YOUR_CLIENT_SECRET",
       redirect_uri="http://127.0.0.1:8888/callback",
       scope="playlist-read-private playlist-read-collaborative"
   ))
   print("Logged in as:", sp.me()["display_name"])
   # Copy the refresh_token from the generated .cache file
Save credentials in Secrets Manager (recommended name: spotify/etl/creds):

json
Copy code
{
  "client_id": "xxxxx",
  "client_secret": "xxxxx",
  "redirect_uri": "http://127.0.0.1:8888/callback",
  "refresh_token": "xxxxx"
}
If you only use client-credentials flows, you don’t need a refresh token.

## 🪣 S3 layout

perl

Copy code

s3://<BUCKET>/

  ├─ raw_data/
  
  │  ├─ to_be_processed/  # extractor writes JSON here
  
  │  └─ processed_data/         # transformer moves processed JSON here
  
  └─ transformed_data/
  
     ├─ artists_data/
     
     ├─ album_data/
     
     └─ songs_data/
     
## 📦 Lambda Layer (dependencies)

Build once and attach to both functions:

bash

Copy code

mkdir -p python

pip install -t python spotipy==2.23.0 pandas==2.2.2 requests urllib3 certifi charset-normalizer idna
zip -r9 spotipy_pandas_layer.zip python

Upload as a Lambda Layer (runtime: Python 3.11) and attach it.

## 🧱 IAM (minimal policy)

Attach to both Lambdas (replace YOUR_BUCKET):

json

Copy code
{
  "Version": "2012-10-17",
  "Statement": [
    {"Effect":"Allow","Action":["logs:CreateLogGroup","logs:CreateLogStream","logs:PutLogEvents"],"Resource":"*"},
    {"Effect":"Allow","Action":["s3:GetObject","s3:PutObject"],"Resource":[
      "arn:aws:s3:::YOUR_BUCKET/raw_data/*",
      "arn:aws:s3:::YOUR_BUCKET/transformed_data/*"
    ]},
    {"Effect":"Allow","Action":["s3:ListBucket"],"Resource":"arn:aws:s3:::YOUR_BUCKET"},
    {"Effect":"Allow","Action":["secretsmanager:GetSecretValue"],"Resource":"*"}
  ]
}

## ⚙️ Lambda configuration

-data_extraction — ENV VARS

-Name	Example/Notes

-BUCKET_NAME	spotify-etl-pipeline-hp (or your bucket)

-RAW_PREFIX	raw_data/to_be_processed/

-CLIENT_ID	Only if using client-credentials

-CLIENT_SECRET	Only if using client-credentials

-SPOTIFY_SECRET_ID	Secrets Manager id with refresh token JSON (opt.)

 Use SpotifyClientCredentials for public endpoints.

For user endpoints, initialize SpotifyOAuth with credentials read from Secrets Manager (refresh-token flow).

data_transformation — ENV VARS

Name	Example/Notes

BUCKET_NAME	same as above

RAW_PREFIX	raw_data/to_be_processed/

PROCESSED_PREFIX	raw_data/processed_data/

TGT_PREFIX	transformed_data/

Trigger: S3 ObjectCreated on RAW_PREFIX.

Important normalization (release dates):

python

Copy code
# album_df columns: release_date, release_date_precision ∈ {"year","month","day"}

def _parse_release_date(date_str, precision):

    import pandas as pd
    
    if pd.isna(date_str): return pd.NaT
    
    if precision == "year":  return pd.to_datetime(f"{date_str}-01-01", errors="coerce")
    
    if precision == "month": return pd.to_datetime(f"{date_str}-01",    errors="coerce")
    
    return pd.to_datetime(date_str, errors="coerce")
    
## ⏰ Scheduling & events

EventBridge rule: rate(1 day) → target data_extraction

S3 Notification: ObjectCreated on raw_data/to_be_processed/ → target data_transformation


## 🧪 Notes & gotchas

Playlist 404s: Some chart playlists aren’t API-exposed in every region. Use Search API to resolve IDs, or snapshot another public editorial list daily and aggregate weekly.

OAuth errors after consent: usually redirect URI mismatch or incorrect client secret. Delete .cache locally and re-auth.

Lambda import errors: Your layer zip must contain a top-level python/ folder and match the Lambda runtime version.

S3 key safety: Avoid : in filenames when composing timestamps.

## 🔒 Security

Rotate your Spotify client secret if it’s ever exposed.

Store secrets (client id/secret, refresh token) in AWS Secrets Manager.

Least-privilege IAM policies; separate roles per Lambda when possible.

## 📜 License

MIT (or update to your preferred license).

makefile

Copy code

::contentReference[oaicite:0]{index=0}
