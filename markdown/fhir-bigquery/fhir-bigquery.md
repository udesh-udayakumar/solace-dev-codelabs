author: Udesh Udayakumar
summary: Ingest FHIR (Fast Healthcare Interoperability Resources) to BigQuery
id: fhir-bigquery
tags:
categories: Codelabs
environments: Web
status: Published
feedback link: https://github.com/SolaceDev/solace-dev-codelabs/tree/master/markdown/fhir-bigquery

# Ingest FHIR (Fast Healthcare Interoperability Resources) to BigQuery

## Introduction

Duration: 0:01:00

![image_caption](img/fhir.png)

This codelab demonstrates a data ingestion pattern to ingest FHIR R4 formatted healthcare data (Regular Resources) into BigQuery using Cloud Healthcare FHIR APIs. Realistic healthcare test data has been generated and made available in the Google Cloud Storage bucket (gs://testing-demo-data-udesh/) for you.

#### In this code lab you will learn:

- How to import FHIR R4 resources from GCS into Cloud Healthcare FHIR Store.
- How to export FHIR data from FHIR Store to a Dataset in BigQuery.
- What do you need to run this demo?

- You need access to a GCP Project.
- You must be assigned an Owner role to the GCP Project.
- FHIR R4 resources in NDJSON format (content-structure=RESOURCE)
- If you don't have a GCP Project, follow [these steps](https://cloud.google.com/resource-manager/docs/creating-managing-projects) to create a new GCP Project.

FHIR R4 resources in NDJSON format has been pre-loaded into GCS bucket at the following locations:

- **gs://testing-demo-data-udesh/fhir_r4_ndjson/** - Regular Resources

All of the resources above have [new line delimiter JSON (NDJSON)](https://www.hl7.org/fhir/nd-json.html) file format but different content structure:
- [Regular Resources](https://github.com/synthetichealth/synthea/wiki/HL7-FHIR) in ndjson format - each line in the file contains a core FHIR resource in JSON format (like [Patient](https://www.hl7.org/fhir/patient.html), [Observation](https://www.hl7.org/fhir/observation.html), etc). Each ndjson file contains FHIR resources of the same resource type. For example Patient.ndjson will contain one or more FHIR resources of resourceType = ‘ [Patient](https://www.hl7.org/fhir/patient.html)' and Observation.ndjson will contain one or more FHIR resources of resourceType = ‘ [Observation](https://www.hl7.org/fhir/observation.html)'.


If you need a new dataset, you can always generate it using [SyntheaTM](https://github.com/synthetichealth/synthea). Then, upload it to GCS instead of using the bucket provided in codelab.

## Environment Setup

Duration: 0:04:00

Follow these steps to enable Healthcare API and grant required permissions:

### Initialize shell variables for your environment

To find the PROJECT_NUMBER and PROJECT_ID, refer to [Identifying projects](https://cloud.google.com/resource-manager/docs/creating-managing-projects#identifying_projects).

```bash
<!-- CODELAB: Initialize shell variables -->
export PROJECT_ID=<PROJECT_ID>
export PROJECT_NUMBER=<PROJECT_NUMBER>
export SRC_BUCKET_NAME=testing-demo-data-udesh
export BUCKET_NAME=<BUCKET_NAME>
export DATASET_ID=<DATASET_ID>
export FHIR_STORE=<FHIR_STORE>
export BQ_DATASET=<BQ_DATASET>
```

### [Enable Healthcare API](https://cloud.google.com/apis/docs/getting-started#enabling_apis)

Following steps will enable Healthcare APIs in your GCP Project. It will add Healthcare API [service account](https://cloud.google.com/iam/docs/service-accounts) to the project.

1. Go to the [GCP Console API Library](https://console.cloud.google.com/apis/library?project=_).
2. From the projects list, select your project.
3. In the API Library, select the API you want to enable. If you need help finding the API, use the search field and the filters.
4. On the API page, click **ENABLE**.

### Create a Google Cloud Storage bucket in your GCP Project

```bash
gsutil mb gs://$BUCKET_NAME
```

### Copy synthetic data to your GCP Project

```bash
gsutil -m cp gs://$SRC_BUCKET_NAME/fhir_r4_ndjson/**.ndjson \
gs://$BUCKET_NAME/fhir_r4_ndjson/
```

### Grant Permissions
Before importing FHIR resources from Cloud Storage and exporting to BigQuery, you must grant additional permissions to the Cloud Healthcare Service Agent [service account](https://cloud.google.com/iam/docs/service-accounts). For more information, see [FHIR store Cloud Storage](https://cloud.google.com/healthcare-api/docs/permissions-healthcare-api-gcp-products#fhir_store_cloud_storage_permissions) and [FHIR store BigQuery permissions](https://cloud.google.com/healthcare-api/docs/permissions-healthcare-api-gcp-products#dicom_store_bigquery_permissions).

#### Grant Storage Object Admin Permission
```bash
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member=serviceAccount:service-$PROJECT_NUMBER@gcp-sa-healthcare.iam.gserviceaccount.com \
  --role=roles/storage.objectAdmin
```

#### Grant BigQuery Admin Permissions
```bash
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member=serviceAccount:service-$PROJECT_NUMBER@gcp-sa-healthcare.iam.gserviceaccount.com \
  --role=roles/bigquery.admin
```


## Environment Setup

Duration: 0:04:00

Follow these steps to ingest data from NDJSON files to healthcare dataset in BigQuery using Cloud Healthcare FHIR APIs:

### Create Healthcare Dataset and FHIR Store

#### Create Healthcare dataset using [Cloud Healthcare APIs](https://cloud.google.com/sdk/gcloud/reference/beta/healthcare/datasets/create)

```bash
gcloud beta healthcare datasets create $DATASET_ID --location=us-central1
```

#### Create FHIR Store in dataset using [Cloud Healthcare APIs](https://cloud.google.com/sdk/gcloud/reference/beta/healthcare/datasets/create)

```bash
gcloud beta healthcare fhir-stores create $FHIR_STORE \
  --dataset=$DATASET_ID --location=us-central1 --version=r4
```

*Note: GCP Healthcare API currently supports the following [locations](https://cloud.google.com/healthcare-api/docs/regions). We will use us-central1 for our convenience.*

## Import data into FHIR Store

### Import test data from [Google Cloud Storage to FHIR Store](https://cloud.google.com/sdk/gcloud/reference/beta/healthcare/fhir-stores/import/gcs).

We will use preloaded files from GCS Bucket. These files contain FHIR R4 regular resources in the NDJSON format. As a response, you will get OPERATION_NUMBER, which can be used in the validation step.

#### Import Regular Resources from the GCS Bucket in your GCP Project

```bash
gcloud beta healthcare fhir-stores import gcs $FHIR_STORE \
  --dataset=$DATASET_ID --async \
  --gcs-uri=gs://$BUCKET_NAME/fhir_r4_ndjson/**.ndjson \
  --location=us-central1 --content-structure=RESOURCE
```

*Remember: You can always generate a new set of realistic test data using [SyntheaTM](https://github.com/synthetichealth/synthea) for this lab. For your ease we are providing a dataset in GCS: gs://testing-demo-data-udesh/*

### Validate

The validate operation finished successfully. It might take a few minutes of operation to finish, so you might need to repeat this command a few times with some delay.

```bash
gcloud beta healthcare operations describe OPERATION_NUMBER \
  --dataset=$DATASET_ID --location=us-central1
```

## Export data from FHIR Store to BigQuery

### [Create a BigQuery Dataset](https://cloud.google.com/sdk/gcloud/reference/alpha/healthcare/fhir-stores/create)

```bash
bq mk --location=us --dataset $PROJECT_ID:$BQ_DATASET
```

### Export healthcare data from [FHIR Store to BigQuery Dataset](https://cloud.google.com/sdk/gcloud/reference/beta/healthcare/fhir-stores/export/bq)

```bash
gcloud beta healthcare fhir-stores export bq $FHIR_STORE \
  --dataset=$DATASET_ID --location=us-central1 --async \
  --bq-dataset=bq://$PROJECT_ID.$BQ_DATASET \
  --schema-type=analytics
```

As a response, you will get OPERATION_NUMBER, which can be used in the validation step.

### Validate

#### Validate operation finished successfully

```bash
gcloud beta healthcare operations describe OPERATION_NUMBER \
  --dataset=$DATASET_ID --location=us-central1
```

#### Validate if BigQuery Dataset has all 16 tables

```bash
bq ls $PROJECT_ID:$BQ_DATASET
```

## Clean up

To avoid incurring charges to your Google Cloud Platform account for the resources used in this tutorial, you can clean up the resources that you created on GCP so they won't take up your quota, and you won't be billed for them in the future. The following sections describe how to delete or turn off these resources.

### Delete the project

The easiest way to eliminate billing is to delete the project that you created for the tutorial.

To delete the project:

Negative
: *Caution: Deleting a project has the following effects:*
*Everything in the project is deleted. If you used an existing project for this tutorial, when you delete it, you also delete any other work you've done in the project.*
*Custom project IDs are lost. When you created this project, you might have created a custom project ID that you want to use in the future. To preserve the URLs that use the project ID, such as an appspot.com URL, delete selected resources inside the project instead of deleting the whole project.*
*If you plan to explore multiple tutorials and quickstarts, reusing projects can help you avoid exceeding project quota limits.*

1. In the GCP Console, go to the [Projects page](https://console.cloud.google.com/cloud-resource-manager).
2. In the project list, select the project you want to delete and click **Delete**.
3. In the dialog, type the project ID, and then click **Shut down** to delete the project.

*If you need to keep the project, you can delete the Cloud healthcare dataset and BigQuery dataset using the following instructions.*

### Delete the Cloud Healthcare API dataset

Follow the [steps](https://cloud.google.com/healthcare-api/docs/datasets#deleting_a_dataset) to delete Healthcare API dataset using both GCP Console and gcloud CLI.

Quick CLI command:

```bash
gcloud beta healthcare datasets delete $DATASET_ID --location=us-central1
```

### Delete the BigQuery dataset

Follow the [steps](https://cloud.google.com/healthcare-api/docs/datasets#deleting_a_dataset) to delete BigQuery dataset using different interfaces.

Quick CLI command:

```bash
bq rm -r -f $PROJECT_ID:$DATASET_ID
```

Conclusion

- You've successfully completed the code lab to ingest healthcare data in BigQuery using Cloud Healthcare APIs.

- You imported FHIR R4 compliant synthetic data from Google Cloud Storage into the Cloud Healthcare FHIR APIs.

- You exported data from the Cloud Healthcare FHIR APIs to BigQuery.

- You now know the key steps required to start your Healthcare Data Analytics journey with BigQuery on Google Cloud Platform.