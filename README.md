# Azure-storage-optimization-task
This repository deals with handling optimization tasks related to billing records in Azure Infrastructure

# Key Components:

**Azure Cosmos DB**: Continues to store recent (less than three months old) billing records for fast read access.
**Azure Blob Storage**  Used as the archival tier for storing older billing records in a cost-optimized storage tier (Cool or Archive, with Cool recommended for your latency requirements).
**Azure Functions (or Azure Logic Apps):** An automated, serverless process to periodically identify and move older records from Cosmos DB to Blob Storage.
**API Gateway/Backend Service (Azure Functions/API Management):** The existing service handling read requests will be modified to check Blob Storage if the record isn't found in Cosmos DB.

# Cost Optimization Strategies:

**Azure Blob Storage Tiering:** Utilize the "Cool" storage tier, which offers lower storage costs compared to "Hot" and is suitable for infrequently accessed data with acceptable access latency (in the order of seconds). The "Archive" tier has even lower costs but higher retrieval latency, potentially not meeting your requirements.
**Cosmos DB Throughput Optimization:** As the volume of data in Cosmos DB reduces, you can potentially scale down the provisioned throughput (RU/s) for the collection, leading to significant cost savings. Monitor your RU/s utilization after implementing the archival process.
**Serverless Compute for Archival:** Azure Functions or Logic Apps provide a cost-effective way to automate the archival process, as you only pay for the execution time.

# Simplicity & Ease of Implementation:

Leveraging managed Azure services like Blob Storage and Functions simplifies infrastructure management.
The archival logic can be implemented with relatively straightforward code.
No changes are required to the core Cosmos DB schema or the existing API contracts.
**No Data Loss & No Downtime:**
Data is moved to Blob Storage after it's successfully written to Cosmos DB initially, ensuring no data loss.
The archival process can run in the background without impacting the availability of recent data in Cosmos DB.
The enhanced read logic in the API Gateway/Backend Service ensures that old data is still accessible seamlessly through the existing API.
**No Changes to API Contracts:**
The existing read and write API endpoints for billing records remain unchanged. The underlying data retrieval mechanism is modified within the Backend Service.

# Architecture 


                       +----------------------+
                        |  Client Application  |
                        +----------+-----------+
                                   |
                                   v
                        +----------+-----------+
                        | Billing API (Unchanged) |
                        +----------+-----------+
                                   |
                    +--------------------------+
                    | Backend Service / API Gateway |
                    +----------+-----------+
                           /                \
                          /                  \
                         v                    v
            +---------------------+    +--------------------------+
            |  Azure Cosmos DB    |    | Azure Blob Storage (Cool)|
            | (Recent - < 3 Months)|    | (Archived - > 3 Months)  |
            +----------+----------+    +----------+----------+
                       |                              ^
                       |                              |
                       +------------------------------+
                              |
                              | Scheduled Trigger (Time-Based)
                              v
            +-------------------------------------+
            | Azure Function (Archival Process) |
            +-------------------------------------+
               (Queries Cosmos DB for old data,
                Serializes to JSON, uploads to Blob,
                Deletes from Cosmos DB)






# Implementation Details :

Azure Function (Python Example for Archival):
 ```sh
 
import azure.functions as func
import azure.cosmos.cosmos_client as cosmos_client
import azure.storage.blob as blob_service
import json
import os
from datetime import datetime, timedelta

Cosmos DB Configuration
COSMOS_ENDPOINT = os.environ["COSMOS_ENDPOINT"]
COSMOS_KEY = os.environ["COSMOS_KEY"]
COSMOS_DATABASE_NAME = "BillingDB"
COSMOS_CONTAINER_NAME = "BillingRecords"

Blob Storage Configuration
BLOB_CONNECTION_STRING = os.environ["BLOB_CONNECTION_STRING"]
BLOB_CONTAINER_NAME = "archived-billing-records"

def main(mytimer: func.TimerRequest) -> None:
    utc_timestamp = datetime.utcnow().isoformat()

    if mytimer.past_due:
        logging.info('The timer is past due!')

    logging.info('Python timer trigger function ran at %s', utc_timestamp)

    cosmos_client_obj = cosmos_client.CosmosClient(COSMOS_ENDPOINT, COSMOS_KEY)
    database = cosmos_client_obj.get_database_client(COSMOS_DATABASE_NAME)
    container = database.get_container_client(COSMOS_CONTAINER_NAME)

    blob_service_client = blob_service.BlobServiceClient.from_connection_string(BLOB_CONNECTION_STRING)
    blob_container_client = blob_service_client.get_container_client(BLOB_CONTAINER_NAME)

    cutoff_date = datetime.utcnow() - timedelta(days=90) # 3 months
    cutoff_timestamp = cutoff_date.isoformat()

    query = f"SELECT * FROM c WHERE c._ts < DateTimeToTimestamp('{cutoff_timestamp}') / 1000" # Query based on _ts (timestamp)

    items = list(container.query_items(query=query, enable_cross_partition_query=True))

    for item in items:
        record_id = item.get("id")
        if record_id:
            blob_name = f"{record_id}.json"
            blob_client = blob_container_client.get_blob_client(blob_name)
            try:
                blob_client.upload_blob(json.dumps(item).encode('utf-8'), overwrite=True)
                container.delete_item(item=record_id, partition_key=item.get("partitionKey")) # Assuming you have a partition key
                logging.info(f"Archived record: {record_id} to Blob Storage")
            except Exception as e:
                logging.error(f"Error archiving record {record_id}: {e}")

     if __name__ == "__main__":
    # Example local execution (replace with your actual environment variables)
    os.environ["COSMOS_ENDPOINT"] = "<your_cosmos_endpoint>"
    os.environ["COSMOS_KEY"] = "<your_cosmos_key>"
    os.environ["BLOB_CONNECTION_STRING"] = "<your_blob_connection_string>"
    main(None)
    
 ```






    
