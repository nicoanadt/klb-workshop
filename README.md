# Kalbe workshop

## 1. Copy the sample data to your account

### Allow S3 Access

1. Create S3 bucket

     `s3://[BUCKET-NAME]`, for example `s3://yourname-analytics-workshop-bucket`
     
2. Create folder inside: 

    `s3://[BUCKET-NAME]/data`
    
    `s3://[BUCKET-NAME]/data/raw`
    
    `s3:/[BUCKETNAME]/data/raw/sales_consolidate/`
    

3. Create inline policy for `TeamRole` role in your account

    Open `IAM` > `Role` > Select `TeamRole` > Create Inline Policy
    
    Create new inline policy:
  
    ```
    {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Allow",
                "Action": [
                    "s3:GetObject",
                    "s3:GetObjectAcl",
                    "s3:GetObjectVersion",
                    "s3:GetObjectVersionAcl",
                    "s3:GetObjectVersionTagging",
                    "s3:PutObject",
                    "s3:PutObjectAcl",
                    "s3:PutObjectVersionAcl",
                    "s3:ListBucket",
                    "S3:GetObjectTagging",
                    "S3:PutObjectTagging"
                ],
                "Resource": [
                    "arn:aws:s3:::kalbe-workshop-test-130835040051-apse1",
                    "arn:aws:s3:::kalbe-workshop-test-130835040051-apse1/*"
                ]
            }
        ]
    }
    ```
    
    Click **Save Changes**

4. Open the following link in new tab : https://s3.console.aws.amazon.com/s3/buckets/kalbe-workshop-test-130835040051-apse1?region=ap-southeast-1&prefix=data/sales_consolidate/&showversions=false 

5. Copy all partitions inside `s3://kalbe-workshop-test-130835040051-apse1/data/sales_consolidate/` to your bucket folder:

    `s3:/[BUCKETNAME]/data/raw/sales_consolidate/`
    
## 2. Run Glue Crawler to load data to Glue Data Catalog

### Create IAM Role

In this step we will navigate to the IAM Console and create a new AWS Glue service role. This allows AWS Glue to access the data stored in S3 and to create the necessary entities in the Glue Data Catalog.

1. Go to **IAM**
2. Click **Create role**
    - Choose the service that will use this role: `Glue`
    - Click Next
    - Search for `AmazonS3FullAccess`
    - Select the entry's checkbox
    - Search for `AWSGlueServiceRole`
    - Select the entry's checkbox
    - Click Next
    - Role name: `AnalyticsworkshopGlueRole`
    - Make sure that only two policies attached to this role (`AmazonS3FullAccess`, `AWSGlueServiceRole`)
3. Click **Create role**

### Create Crawler

1. Open AWS Glue page
2. Click on **Crawlers** > **Create Crawler**
     - Crawler info
          - Crawler name: `AnalyticsworkshopCrawler`
     - Click Next
     - Click Add a data source
          Choose a Data source:
          - Data source: `S3`
     - Leave Network connection - optional as-is
     - Select In this account under Location of S3 data
          - Include S3 path: `s3://yourname-analytics-workshop-bucket/data/raw/sales_consolidate/`
          - Leave Subsequent crawler runs to default selection of Crawl all sub-folders
          - Click Add an S3 data source
          - Select recently added S3 data source under Data Sources
     - Click Next
     - IAM Role
          - Under Existing IAM role, select `AnalyticsworkshopGlueRole`
          - Leave everything else as-is.
     - Click Next
     - Output configuration:
          - Click Add database to bring up a new window for creating a database.
          - Database details
          - Name: `klb_db`
          - Click Create database
          - Closes the current window and returns to the previous window.
          - Refresh by clicking the refresh icon to the right of the Target database
          - Choose `klb_db` under Target database
          - Click **Advanced options**
          - Checklist `Update all new and existing partitions with metadata from the table`
     - Under Crawler schedule
          - Frequency: `On demand`
          - Click Next
     - Review all settings under Review and create
     - Click **Create crawler**
3. You should see this message: The following crawler is now created: "AnalyticsworkshopCrawler". 
   - Click **Run crawler** to run the crawler for the first time
   - Wait for few minutes

4. Verify newly created tables in catalog.
     - Navigate to **Glue Catalog** and explore the crawled data:
          - Click database`klb_db`
          - Click Tables in `klb_db`
          - Click `sales_consolidate`
               - Look around and explore the schema for your dataset
               - look for the averageRecordSize, recordCount, compressionType

## 3. Query and analyze the data lake using Amazon Athena

1. Open Athena page > klik **Launch Query Editor**
2. Set query results location in S3 for first time use: 
     - Click **Edit settings**
     - Point to `s3://yourname-analytics-workshop-bucket/query_result/`
     - Click **Save**
4. Explore metadata, and run sample query:  

     ```
     SELECT * FROM "klb_db"."sales_consolidate" limit 10; 
     ```
     
     Explore your data.
6. Create new table using CTAS statement in Athena to create new table with Parquet file

     ```
     CREATE TABLE "klb_db"."sales_consolidate_parquet"
     WITH (
           format = 'Parquet',
           write_compression = 'SNAPPY',
           external_location = 's3://[BUCKETNAME]/data/raw/sales_consolidate_parquet/',
           partitioned_by = ARRAY['filename']) 
           )
     AS SELECT * FROM "klb_db"."sales_consolidate";
     ```
7. Open the S3 location in `s3://[BUCKETNAME]/data/raw/sales_consolidate_parquet/`
     - Observe the file format
     - Compare the total filesize of csv file vs parquet


## 4. Load to Redshift using Redshift Spectrum

### Query in Redshift

1. **Open Redshift Query Editor v2**
     - If prompted, you may need to configure the Query Editor.
     - On the left-hand side, click on the Redshift environment you want to connect to.
     ![](https://static.us-east-1.prod.workshops.aws/public/36b90137-7dd0-42c6-b8f2-000e56e508fc/static/images/lab1/ConnectionV2.png)
2. Connect to `consumercluster-xxxxxx`
     - Enter the Database name and user name. Click connect. These credentials should be used for both the Serverless endpoint (workgroup-xxxxxxx) as well as the provisioned cluster (consumercluster-xxxxxxxxxx).
     ```
     Username `awsuser`
     Password `Awsuser123`
     ```
3. Select a sample query. If it is successful, congrats! You are now connected to Redshift

     ```
     select * from pg_user;
     ```

4.  Create schema

     ```
     CREATE SCHEMA klb_rs;
     ```


### Create external schema to use Redshift Spectrum

1. In Redshift, there are two ways of loading data from S3 to Redshift using Redshift features:
     - Using COPY command to load data from S3 files
     - Using Redshift Spectrum to query into S3 data lake
     
     
     In this exercise we will explore Redshift Spectrum to automatically create the table for us based on the crawler object.

2. Create external schema in Redshift

     ```
     create external schema klb_spectrum 
     from data catalog 
     database 'klb_db' 
     iam_role 'arn:aws:iam::130835040051:role/myspectrum_role';
     ```

3. Query to S3 data lake using Redshift Spectrum
4. Create Redshift table from existing Glue Data Catalog table
5. Load data using Redshift Spectrum into Redshift internal table
6. Observe the runtime of data load
     - Observe the performance
     - Observe the total cost of data loading process



## 5. Load to Redshift using Glue Studio

### Setup S3 Gateway Endpoint in VPC

In this step, we will create S3 Gateway Endpoint so that Redshift cluster can communicate with S3 using its private IP.

1. Go to: **AWS VPC Console**
2. Click Create endpoint
     - Name tag - optional: `RedshiftS3EP`
     - Select AWS Services under Service category (which is the default selection)
     - Under Service name search box, search for "s3" and hit enter/return.
     - `com.amazonaws.[region].s3` should come up as search result. Select this option with type as `Gateway`.
     - Under VPC, choose non-default VPC. This is the same VPC which was used for configuring redshift cluster.
          - If you have more than VPC listed in the drop down list, double check Redshift VPC to avoid any confusion. Do the following:
          - Go to: Redshift Console  
          - Click on redshift cluster name
          - Click on Properties tab.
          - Scroll down and check Network and security section for VPC name.
     - Once you have double checked VPC id, move to configuring Route tables section.
     - Select the listed route tables (checklist both route table)
3. Click Create endpoint. It should take a couple of seconds to provision this. Once this is ready, you should see Status as Available against the newly created S3 endpoint.

### Setup Glue Connection

1. Open AWS Glue page
2. Open `Data Connections page, create connection in Glue by clicking **Create Connection**
     - Enter connection name `redshift-cluster-connection-dev`
     - Choose connection type `Redshift`
     - Choose Database instances `consumercluster-xxxxxxx`
     - Database name `dev`
     - Username `awsuser`
     - Password `Awsuser123`
     - Click **Create Connection**

### Create Glue Job
1. Open **ETL Jobs** page
2. Choose **Visual with a source and target**
3. Choose Source `AWS Glue Data Catalog` and Target `Amazon Redshift`
4. Click **Create** on upper right
5. Glue Studio Editor page is opened.
     - Choose Data source 
          - Database: `klb_db`
          - Table: `Sales_consolidate`
     - Choose Data target
          - Choose **Direct data connection**
          - Select Redshift connection `redshift-cluster-connection-dev`
          - Select Schema `klb_rs`
          - Select new table name `sales_consolidate_rs`
     - Save job as `klb_sales_consolidate_s3_to_rs`
     - Click Job Details
          - Choose IAM Role `AnalyticsworkshopGlueRole`
          - Enable `Automatically scale the number of workers`
     - Click **Save**
     - Click **Run**
6. Once the job is finished, note down the:
     - Execution runtime
     - Number of workers
     - DPU hours : used for cost calculation


## Orchestrate Glue jobs using Step Function
## Orchestrate Redshift queries using Step Function
