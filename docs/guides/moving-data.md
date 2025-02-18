---
description: Moving data to and from cloud providers
tags:
    - data
    - storage
    - read data
    - write data
    - cloud providers
    - AWS
    - S3
    - Azure Storage
    - Azure Blob Storage
    - Azure
    - GCP
    - Google Cloud Storage
    - GCS
    - moving data
search:
  boost: 2
---

# Read and Write Data to and From Cloud Provider Storage

Writing data to cloud-based storage and reading data from that storage is a common task in data engineering.
In this guide we'll learn how to use Prefect to move data to and from AWS, Azure, and GCP blob storage.

## Prerequisites

- Prefect [installed](/getting-started/installation/)
- Authenticated with [Prefect Cloud](/cloud/cloud-quickstart/) (or self-hosted [Prefect server](/guides/host/) instance)
- A cloud provider account (e.g. [AWS](https://aws.amazon.com/))

## Install relevant Prefect integration library

In the CLI, install the Prefect integration library for your cloud provider:

=== "AWS"

    [prefect-aws](https://prefecthq.github.io/prefect-aws/) provides blocks for interacting with AWS services.

    ```bash
    pip install -U prefect-aws
    ```

=== "Azure"

    [prefect-azure](https://prefecthq.github.io/prefect-azure/) provides blocks for interacting with Azure services.

    ```bash
     pip install -U prefect-azure
    ```

=== "GCP"

    [prefect-gcp](https://prefecthq.github.io/prefect-gcp/) provides blocks for interacting with GCP services.
    
    ```bash
     pip install -U prefect-gcp
    ```

## Register the block types

Register the new block types with Prefect Cloud (or with your self-hosted Prefect server instance):

=== "AWS"

    ```bash
    prefect block register -m prefect_aws  
    ```

=== "Azure"

    ```bash
    prefect block register -m prefect_azure 
    ```

=== "GCP"

    ```bash
    prefect block register -m prefect_gcp
    ```

We should see a message in the CLI that several block types were registered.
If we check the UI, we should see the new block types listed.

## Create a storage bucket

Create a storage bucket in the cloud provider account.
Ensure the bucket is publicly accessible or create a user or service account with the appropriate permissions to fetch and write data to the bucket.

## Create a credentials block

If the bucket is private, there are several options to authenticate:

1. At deployment runtime, ensure the runtime environment is authenticated.
2. Create a block with configuration details and reference it when creating the storage block.

If saving credential details in a block we can use a credentials block specific to the cloud provider or use a more generic secret block.
We can create [blocks](/concepts/blocks/) via the UI or Python code.
Below we'll use Python code to create a credentials block for our cloud provider.

!!! Warning "Credentials safety"

    Reminder, don't store credential values in public locations such as public git platform repositories. 
    In the examples below we use environment variables to store credential values.

=== "AWS"

    ```python
    import os
    from prefect_aws import AwsCredentials

    my_aws_creds = AwsCredentials(
        aws_access_key_id="123abc",
        aws_secret_access_key=os.environ.get("MY_AWS_SECRET_ACCESS_KEY"),
    )
    my_aws_creds.save(name="my-aws-creds-block", overwrite=True)
    ```

=== "Azure"

    ```python
    import os
    from prefect_azure import AzureBlobStorageCredentials

    my_azure_creds = AzureBlobStorageCredentials(
        connection_string=os.environ.get("MY_AZURE_CONNECTION_STRING"),
    )
    my_azure_creds.save(name="my-azure-creds-block", overwrite=True)
    ```

=== "GCP"

    We recommend specifying the service account key file contents as a string, rather than the path to the file, because that file might not be available in your production environments.

    ```python
    import os
    from prefect_gcp import GCPCredentials

    my_gcp_creds = GCPCredentials(
        service_account_info=os.environ.get("GCP_SERVICE_ACCOUNT_KEY_FILE_CONTENTS"), 
    )
    my_gcp_creds.save(name="my-gcp-creds-block", overwrite=True)
    ```

Run the code to create the block. We should see a message that the block was created.

## Create a storage block

Let's create a block for the chosen cloud provider using Python code or the UI.
In this example we'll use Python code.

=== "AWS"

    Note that the `S3Bucket` block is not the same as the `S3` block that ships with Prefect. 
    The `S3Bucket` block we use in this example is part of the `prefect-aws` library and provides additional functionality. 

    We'll reference the credentials block created above.

    ```python
    from prefect_aws import S3Bucket

    s3bucket = S3Bucket.create(
        bucket="my-bucket-name",
        credentials="my-aws-creds-block"
        )
    s3bucket.save(name="my-s3-bucket-block", overwrite=True)
    ```

=== "Azure"

    Note that the `AzureBlobStorageCredentials` block is not the same as the Azure block that ships with Prefect. 
    The `AzureBlobStorageCredentials` block we use in this example is part of the `prefect-azure` library and provides additional functionality. 

    Azure blob storage doesn't require a separate block, the connection string used in the `AzureBlobStorageCredentials` block can encode the information needed. 

=== "GCP"

    Note that the `GcsBucket` block is not the same as the `GCS` block that ships with Prefect. 
    The `GcsBucket` block is part of the `prefect-gcp` library and provides additional functionality. 
    We'll use it here.

    We'll reference the credentials block created above.

    ```python
    from prefect_gcp.cloud_storage import GcsBucket

    gcsbucket = GcsBucket(
        bucket="my-bucket-name", 
        credentials="my-gcp-creds-block"
        )
    gcsbucket.save(name="my-gcs-bucket-block", overwrite=True)
    ```

Run the code to create the block. We should see a message that the block was created.

## Write data

Use your new block inside a flow to write data to your cloud provider.

=== "AWS"

    ```python
    from pathlib import Path
    from prefect import flow
    from prefect_aws.s3 import S3Bucket

    @flow()
    def upload_to_s3():
        """Flow function to upload data"""
        path = Path("my_path_to/my_file.parquet")
        aws_block = S3Bucket.load("my-s3-bucket-block")
        aws_block.upload_from_path(from_path=path, to_path=path)

    if __name__ == "__main__":
        upload_to_s3()
    ```

=== "Azure"

    ```python
    from prefect import flow
    from prefect_azure import AzureBlobStorageCredentials
    from prefect_azure.blob_storage import blob_storage_upload

    @flow
    def upload_to_azure():
        """Flow function to upload data"""
        blob_storage_credentials = AzureBlobStorageCredentials.load(
            name="my-azure-creds-block"
        )

        with open("my_path_to/my_file.parquet", "rb") as f:
            blob_storage_upload(
                data=f.read(),
                container="my_container",
                blob="my_path_to/my_file.parquet",
                blob_storage_credentials=blob_storage_credentials,
            )

    if __name__ == "__main__":
        upload_to_azure()
    ```

=== "GCP"

    ```python
    from pathlib import Path
    from prefect import flow
    from prefect_gcp.cloud_storage import GcsBucket

    @flow()
    def upload_to_gcs():
        """Flow function to upload data"""
        path = Path("my_path_to/my_file.parquet")
        gcs_block = GcsBucket.load("my-gcs-bucket-block")
        gcs_block.upload_from_path(from_path=path, to_path=path)

    if __name__ == "__main__":
        upload_to_gcs()
    ```

## Read data

Use your block to read data from your cloud provider inside a flow.

=== "AWS"

    ```python

    from prefect import flow
    from prefect_aws import S3Bucket

    @flow
    def download_from_s3():
        """Flow function to download data"""
        s3_block = S3Bucket.load("my-s3-bucket-block")
        s3_block.get_directory(
            from_path="my_path_to/my_file.parquet", 
            local_path="my_path_to/my_file.parquet"
        )

    if __name__ == "__main__":
        download_from_s3()
    ```

=== "Azure"

    ```python
    from prefect import flow
    from prefect_azure import AzureBlobStorageCredentials
    from prefect_azure.blob_storage import blob_storage_download

    @flow
    def download_from_azure():
        """Flow function to download data"""
        blob_storage_credentials = AzureBlobStorageCredentials.load(
            name="my-azure-creds-block"
        )
        blob_storage_download(
            blob="my_path_to/my_file.parquet",
            container="my_container",
            blob_storage_credentials=blob_storage_credentials,
        )

    if __name__ == "__main__":
        download_from_azure()
    ```

=== "GCP"

    ```python
    from prefect import flow
    from prefect_gcp.cloud_storage import GcsBucket

    @flow
    def download_from_gcs():
        gcs_block = GcsBucket.load("my-gcs-bucket-block")
        gcs_block.get_directory(
            from_path="my_path_to/my_file.parquet", 
            local_path="my_path_to/my_file.parquet"
        )

    if __name__ == "__main__":
        download_from_gcs()

    ```

In this guide we've seen how to use Prefect to read data from and write data to cloud providers!

## Next steps

Check out the [`prefect-aws`](https://prefecthq.github.io/prefect-aws/), [`prefect-azure`](https://prefecthq.github.io/prefect-azure/), and [`prefect-gcp`](https://prefecthq.github.io/prefect-gcp/) docs to see additional methods for interacting with cloud storage providers.
Each library also contains blocks for interacting with other cloud-provider services.
