# Kalbe workshop

## 1. Copy the sample data to your account

### Allow S3 Access

1. Create S3 bucket

     `s3://[BUCKET-NAME]`
     
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

### Create IAM ROle

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
          - Include S3 path: `s3://yourname-analytics-workshop-bucket/data/`
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
          - Choose analyticsworkshopdb under Target database
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

## 3. Query and analyze using Athena
## 4. Load to Redshift using Glue Studio
## 5. Load to Redshift using Redshift Spectrum
## Query in Redshift
## Orchestrate Glue jobs using Step Function
## Orchestrate Redshift queries using Step Function
