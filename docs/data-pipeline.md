# Data Pipeline

Before we start to train the model, we need to clean the data. For training I decided to use AWS and some of its in-built ETL and Data processing systems. This is more aligned to corporate use and allows me to traing on specialized hardware.

## High Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────────────────┐
│                              AWS Data Processing Pipeline                                   │
└─────────────────────────────────────────────────────────────────────────────────────────────┘

┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  MIMIC-IV   │     │     S3      │     │   AWS Glue  │     │     S3      │     │  SageMaker  │
│  Waveform   │────▶│   Bucket    │────▶│   Athena    │────▶│   Bucket    │────▶│  Training   │
│  Database   │     │   (Raw)     │     │  SageMaker  │     │ (Processed) │     │             │
└─────────────┘     └─────────────┘     └─────────────┘     └─────────────┘     └─────────────┘
                           │                   │                   │
                           ▼                   ▼                   ▼
                    ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
                    │  Waveforms  │     │  Labeling   │     │  Windowed   │
                    │  + Numerics │     │  + Features │     │  Samples    │
                    └─────────────┘     └─────────────┘     └─────────────┘
                                               ▲
                                               │
                                        ┌─────────────┐
                                        │  BigQuery   │
                                        │  MIMIC-IV   │
                                        │  Clinical   │
                                        └─────────────┘
```

_This is first pass example on the pipeline_

## Terraform

We will be using terraform to provision our pipeline. This allows anyone who looks at this project to be able to recreate it for themselves. _Note:_ There may be some Clickops involved, but i will try my best to keep it logged.

## Provision Notes through Clickops

1. Create Bucket for Raw Data
2. Use wget (potentially through an EC2 Bucket to get the data in there)
3. Setup AWS Glue to read .csv.gz files from the bucket, and peform necessary transformations
4. Perform BigQuery calls to get patient death data and / or if they had a cardiac event. Coalesc.
5. Move data from Glue to an Output bucket.

(When we start using Waveform Data)

1. Get data from the Bucket (.dat and .header)
2. Use Sagemaker or EC2 to clean the data
3. Send to the output bucket

## Notes

From the dataset, we have compressed csv's for numerical data, and .dat with a corresponding .header file to transform. For the pipeline, it will intially just have the S3 bucket, then AWS Glue(?) to open the csvs (Potentially could just do this on EC2 with a recursive function). Then Label if needed, and any extrapolation for missing data. Aswell for all this, in order to identify which data has a cardiac event, we will need to use the [MIMIC-IV clinical database](https://physionet.org/content/mimiciv/3.1/) to cross join the data. Then we know that for x patient, they had a cardiac event, and potentially deduce time of event from the data. We may be able to query from Googles BigQuery for the clinical database, saving us the headache.

19/1/26

Going to provision the pipeline on AWS.

1. Provision the S3 Bucket for Raw Data [x]

- Created with the default Settings in ap-southeast-2

2. Download the eICU Data onto the Bucket [x]

- Using ec2 intermediary to download then sync to s3
- Simple EC2 setup, t3.micro with 24GiB gp3 storage
- Ran `wget -r -N -c -np https://physionet.org/files/eicu-crd-demo/2.0.1/` on the EC2 Instance (Switched Datasets initially, will see how it goes. Reason for switching was better numerics)
- Create IAM Role for the EC2 instance for being able to sync to the bucket
- cd into the physioNet directory thats created, then execute `aws s3 sync . s3://cardio-ai-raw-data/eICU`

3. Initialise AWS Glue to process the data from the S3 Bucket [x]

- Need to create a crawler to go through my s3 bucket - Set up the crawler and add the S3 Bucket as the data source - Create IAM role for the crawler to access S3 - Create new Database for the Data (eicu_db) for this one - Set time schedule to on-demand, since i dont have live data at this point - Create and Run - Data ends up in the data catalog for processing - Had issue with: Tables not being in a seperate folder for each, caused issues with table finding.

4. Set up Athena to query and Join the data [ ]
