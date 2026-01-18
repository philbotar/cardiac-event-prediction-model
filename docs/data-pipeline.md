# Data Pipeline

Before we start to train the model, we need to clean the data. For training I decided to use AWS and some of its in-built ETL and Data processing systems. This is more aligned to corporate use and allows me to traing on specialized hardware.

## High Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────────────────┐
│                              AWS Data Processing Pipeline                                   │
└─────────────────────────────────────────────────────────────────────────────────────────────┘

┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  MIMIC-IV   │     │     S3      │     │   AWS Glue  │     │     S3      │     │  SageMaker  │
│  Waveform   │────▶│   Bucket    │────▶│     ETL     │────▶│   Bucket    │────▶│  Training   │
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
