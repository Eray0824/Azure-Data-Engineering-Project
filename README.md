# Earthquake Azure Data Engineering Pipeline: A Comprehensive Guide

## Overview and Architecture

![399150923-bdadd2e0-89be-4683-b53b-fe331be6f6bf](https://github.com/user-attachments/assets/6d2829f4-4454-4b63-a3ae-919f58bfe321)


### Business Case

Earthquake data is incredibly valuable for understanding seismic events and mitigating risks. Government agencies, research institutions, and insurance companies rely on up-to-date information to plan emergency responses and assess risks. With this automated pipeline, we ensure these stakeholders get the latest data in a way that’s easy to understand and ready to use, saving time and improving decision-making.

### Architecture Overview

This pipeline follows a modular architecture, integrating Azure’s powerful data engineering tools to ensure scalability, reliability, and efficiency. The architecture includes:

1. **Data Ingestion**: Azure Data Factory orchestrates the daily ingestion of earthquake data from the USGS Earthquake API.
2. **Data Processing**: Databricks processes raw data into structured formats (bronze, silver, gold tiers).
3. **Data Storage**: Azure Data Lake Storage serves as the backbone for storing and managing data at different stages.
4. **Data Analysis**: Synapse Analytics enables querying and aggregating data for reporting.
5. **Optional Visualization**: Power BI can be used to create interactive dashboards for stakeholders.

### Data Modeling

We implement a **medallion architecture** to structure and organize data effectively:

1. **Bronze Layer**: Raw data ingested directly from the API, stored in Parquet format for future reprocessing if needed.
2. **Silver Layer**: Cleaned and normalized data, removing duplicates and handling missing values, ensuring it’s ready for analytics.
3. **Gold Layer**: Aggregated and enriched data tailored to specific business needs, such as adding in country codes.

### Understanding the API

- The earthquake API provides detailed seismic event data for a specified start and end date.
- **Start Date**: Defines the range of data. This is dynamically set via Azure Data Factory for daily ingestion.
- **API URL**: `https://earthquake.usgs.gov/fdsnws/event/1/`

### Key Benefits

- **Automation**: Eliminates manual data fetching and processing, reducing operational overhead.
- **Scalability**: Handles large volumes of data seamlessly using Azure services.
- **Actionable Insights**: Provides stakeholders with ready-to-use data for informed decision-making.

---


## Step 1: Create a Databricks Resource
1. Create a Databricks resource in Azure.
2. Select the **Standard LTS (Long Term Support)** tier. Avoid using ML or other specialized tiers.

---

## Step 2: Set Up a Storage Account
1. Create a Storage Account and enable **hierarchical namespaces** in the advanced settings.
2. Navigate to the Storage Account resource:
   - Go to **Data Storage > Containers > + Containers**.
   - Create three containers: `bronze`, `silver`, and `gold`.
3. Configure access:
   - Go to **IAM > Add role assignment > Storage Blob Data Contributor**.
   - Click **Next > Managed Identity > Select Members**.
   - Select **Access Connector for Azure Databricks** as the managed identity.

---

## Step 3: Configure Databricks
1. Open the Databricks resource and click **Launch Workspace**.
2. Start a compute instance 
3. Set up external data access:
   - Go to **Catalog > External Data > Credentials > Create Credential**.
   - For the **Access Connector ID**, use the Resource ID of the Access Connector:
     - Search for **Access Connector**, copy the Resource ID, and paste it here.
   - Use this section to grant permissions or delete credentials as needed.
4. Define external locations:
   - Navigate to **External Data > External Locations**.
   - Assign a name, select the storage credential, and specify the URL (use the container name and storage account name for `bronze`, `silver`, and `gold`).

---

## Step 4: Create and Execute Notebooks
1. In the Databricks workspace, create a notebook for each layer (`bronze`, `silver`, `gold`).
   - Execute the notebook and refresh the Storage Account containers to verify updates.
   - Repeat the process for `silver` and `gold` notebooks, adding the corresponding code.

---

## Step 5: Install Required Libraries
1. Before running the `gold` notebook, install the `reverse_geocoder` library.
   - Navigate to **Compute > Cluster > Libraries > + Install New Library**.
   - Select **Source: PyPI** and enter **reverse_geocoder**.
   - Wait a few minutes for the installation to complete.
2. Use cluster-level libraries for consistency and shared environments across notebooks.

---

## Step 6: Optimize Gold Notebook Execution
1. During execution, you may encounter performance bottlenecks with the `reverse_geocoder` Python UDF due to its non-thread-safe nature in distributed setups.
   - Replace the UDF with alternatives like:
     - **Precomputed lookup tables**.
     - **Pandas UDFs for vectorized execution**.
     - **Batch processing geocoding outside Spark**.

---

![data factory2](https://github.com/user-attachments/assets/a2c46f72-1595-4590-9d54-985300d29afa)


## Step 7: Set Up Azure Data Factory (ADF)
1. Create a new Azure Data Factory instance (in a new Resource Group if needed).
2. Launch the ADF studio and create a pipeline:
   - Drag the **Notebook** activity into the pipeline and configure it to run Databricks notebooks.
   - Add a **Databricks Linked Service**:
     - Use the **AutoResolveIntegrationRuntime**.
     - Authenticate with an Access Token (recommended to store the token in a Key Vault for security).
3. Pass parameters to the pipeline:
   - For example, add parameters `start_date` and `end_date` with dynamic values using `@formatDateTime` expressions.
4. Chain notebooks (`bronze`, `silver`, `gold`) to create a pipeline with success dependencies.
5. Validate, publish, and run the pipeline.
6. Schedule the pipeline to run at desired intervals (e.g., daily).

---
![Azure Synapse](https://github.com/user-attachments/assets/d852b130-ca07-498e-8520-d2df1f13103b)

## Step 8: Integrate Azure Synapse Analytics
1. **Create a Synapse Workspace**:
   - Link it to the existing Storage Account.
   - Configure a file system and assign necessary permissions.
2. **Query Data Using Serverless SQL**:
   - Use `OPENROWSET` to query Parquet files stored in `bronze`, `silver`, and `gold` containers.
   - Example query:
     ```sql
     SELECT
         country_code,
         COUNT(CASE WHEN LOWER(sig_class) = 'low' THEN 1 END) AS low_count,
         COUNT(CASE WHEN LOWER(sig_class) IN ('medium', 'moderate') THEN 1 END) AS medium_count,
         COUNT(CASE WHEN LOWER(sig_class) = 'high' THEN 1 END) AS high_count
     FROM
         OPENROWSET(
             BULK 'https://<storage_account>.dfs.core.windows.net/gold/earthquake_events_gold/**',
             FORMAT = 'PARQUET'
         ) AS [result]
     GROUP BY
         country_code;
     ```
3. **Create External Tables** for structured access:
   - Define external tables linked to the `gold` container for better organization and performance.
4. **Optimize Performance**:
   - Use indexing, partitioning, and caching as required.

---

## Optional Step 9: Visualization via Power BI 
---

## Key Considerations
- **Linked Services**: Ensure reusable and secure connections between Azure services.
- **Scalability**: Use Synapse for querying large datasets efficiently.
- **Data Engineering Focus**: Maintain an emphasis on structured pipelines and optimized workflows.

This guide provides a comprehensive approach to setting up a professional-grade Azure Databricks and Synapse workflow for data engineering.
