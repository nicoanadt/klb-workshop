# Kalbe AWS Analytics Workshop

Welcome to the AWS Analytics Workshop. This workshop will go through multiple services of AWS to help you understand how AWS Analytics services can cover your technical and business needs.

![](https://raw.githubusercontent.com/nicoanadt/klb-workshop/main/step-functions/klb-arch-workshop.png)

## 1. Copy the sample data to your account

### 1.1 Allow S3 Access



1. Open `Amazon S3` console page. Create S3 bucket

     `s3://[BUCKET-NAME]`, for example `s3://yourname-analytics-workshop-bucket`
     
2. Create folder inside: 

    `s3://[BUCKET-NAME]/data`
    
    `s3://[BUCKET-NAME]/data/raw`
    
    `s3:/[BUCKETNAME]/data/raw/sales_consolidate/`
    

3. Allow your role `TeamRole` to copy data from source data owner by adding new policies:

    - Open `IAM` console page.
    
    - Click on `Role` > Search for `TeamRole` > Open `TeamRole`
    
    - In TeamRole page, click `Add permissions` > `Create Inline Policy`
    
         - Create new inline policy:

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
                         "arn:aws:s3:::kalbe-workshop-test-130835040051-use1",
                         "arn:aws:s3:::kalbe-workshop-test-130835040051-use1/*"
                     ]
                 }
             ]
         }
         ```
         - Click `Review policy` > Set a name `kalbe-data-access` > Click `Create Policy`
    
    - In TeamRole page, click `Add permissions` > `Attach Policies`
         - Search for `AWSCloudShellFullAccess` then click `Add permissions`
         - Search for `AWSStepFunctionsFullAccess` then click `Add permissions`
    
    
4. Open the following link in new tab to view the data source that we are going to use : https://s3.console.aws.amazon.com/s3/buckets/kalbe-workshop-test-130835040051-use1?region=us-east-1&prefix=data/sales_consolidate/&showversions=false

5. Click on the `CloudShell` button in the lower left side of the screen

6. Copy all partitions inside `s3://kalbe-workshop-test-130835040051-use1/data/sales_consolidate_new/` to your bucket folder:  `s3:/[BUCKETNAME]/data/raw/sales_consolidate/` by running this command in CloudShell. Adjust the `BUCKET-NAME` based as needed.

     ```
     aws s3 cp s3://kalbe-workshop-test-130835040051-use1/data/sales_consolidate_new/ s3://[BUCKET-NAME]/data/raw/sales_consolidate/ --acl bucket-owner-full-control --region us-east-1 --recursive
     ```
7. Observe until all data has been completed and verify in your S3 bucket
    
## 2. Run Glue Crawler to load data to Glue Data Catalog

### 2.1 Create IAM Role

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

### 2.2 Create Crawler

1. Open **AWS Glue** page
2. Click on **Data Catalog** > **Crawlers** > **Create Crawler**
     - Crawler info
          - Crawler name: `AnalyticsworkshopCrawler`
     - Click Next
     - Click **Add a data source**
          - Choose a Data source:
          - Data source: `S3`
     - Leave Network connection - optional as-is
     - Select `In this account` then:
          - Include **S3 path**: `s3://yourname-analytics-workshop-bucket/data/raw/sales_consolidate/`
          - Leave Subsequent crawler runs to default selection,  `Crawl all sub-folders`
          - Click `Add an S3 data source`
          - Select recently added S3 data source under Data Sources
     - Click Next
     - IAM Role
          - Under Existing IAM role, select `AnalyticsworkshopGlueRole`
          - Leave everything else as-is.
     - Click Next
     - Output configuration:
          - Click `Add database` to bring up a new window for creating a database.
               - Database details
               - Name: `klb_db`
               - Click Create database
               - Close the current window and returns to the previous window.
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
           external_location = 's3://[BUCKETNAME]/data/raw/sales_consolidate_parquet/'
           )
     AS SELECT * FROM "klb_db"."sales_consolidate";
     ```
7. Open the S3 location in `s3://[BUCKETNAME]/data/raw/sales_consolidate_parquet/`
     - Observe the file format
     - Compare the total filesize of csv file vs parquet


## 4. Load to Redshift using Redshift Spectrum (Option 1)

In Redshift, there are two ways of loading data from S3 to Redshift using Redshift features:
- Using COPY command to load data from S3 files
- Using Redshift Spectrum to query into S3 data lake
          
In this exercise we will explore Redshift Spectrum to automatically create the table for us based on the glue crawler object.

### 4.1 Query in Redshift

1. Open the **Amazon Redshift** console

2. **Open Redshift Query Editor v2** from the left menu.
     - If prompted, you may need to configure the Query Editor. Click on `Configure Account`
     - On the left-hand side, click on the Redshift environment you want to connect to.
     ![](https://static.us-east-1.prod.workshops.aws/public/36b90137-7dd0-42c6-b8f2-000e56e508fc/static/images/lab1/ConnectionV2.png)
3. Connect to `consumercluster-xxxxxx`
     - Enter the Database name and user name. Click connect. These credentials should be used for both the Serverless endpoint (workgroup-xxxxxxx) as well as the provisioned cluster (consumercluster-xxxxxxxxxx). 
     - **For this exercise, use the provisioned cluster (consumercluster-xxxxxxxxxx)**
     ```
     Username: awsuser
     Password: Awsuser123
     ```
4. Select a sample query. If it is successful, congrats! You are now connected to Redshift

     ```
     select * from pg_user;
     ```
     ![](https://static.us-east-1.prod.workshops.aws/public/36b90137-7dd0-42c6-b8f2-000e56e508fc/static/images/lab1/Users.png)

4.  Create schema

     ```
     CREATE SCHEMA klb;
     ```
     
### 4.2 Create IAM role for Redshift Spectrum

1. Open the IAM console.

2. In the navigation pane, choose **Roles**.

3. Choose **Create role**.

4. Choose AWS service as the trusted entity, and then choose `Redshift` as the use case.

5. Under Use case for other AWS services, choose `Redshift - Customizable` and then choose Next.

6. The Add permissions policy page appears. Choose `AmazonS3ReadOnlyAccess` and `AWSGlueConsoleFullAccess`.
7. For Role name, enter a name for your role, for example `myspectrum_role`.
8. Review the information, and then choose **Create role**.
9. In the navigation pane, choose **Roles**. Choose the name of your new role to view the summary, and then copy the Role ARN to your clipboard. This value is the Amazon Resource Name (ARN) for the role that you just created. You use that value when you create external tables to reference your data files on Amazon S3.

### 4.3 Associate IAM role with cluster

1. Open **Amazon Redshift** console page 

2. On the navigation menu, choose Clusters, then choose the name of the cluster that you want to update (`consumercluster-xxxxxxxxx`)

3. For **Actions**, choose **Manage IAM roles**. The IAM roles page appears.

4. Choose `myspectrum_role` IAM role from the list. Then choose **Associate IAM role** to add it to the list of Attached IAM roles.

5. Choose **Save changes** to associate the IAM role with the cluster. The cluster is modified to complete the change.


### 4.4 Create external schema to use Redshift Spectrum

1. Open **Redshift Query Editor v2**
2.  Create external schema in Redshift

     ```
     create external schema klb_spectrum 
     from data catalog 
     database 'klb_db' 
     iam_role 'arn:aws:iam::[YOUR-ACCOUNT-NUMBER]:role/myspectrum_role';
     ```

3. Query to S3 data lake using Redshift Spectrum

   ```
   select * from klb_spectrum.sales_consolidate_parquet;
   ```

4. Create Redshift internal table from existing Glue Data Catalog table

   ```
   create schema klb_rs;
   
   create table klb_rs.sales_consolidate (like klb_spectrum.sales_consolidate_parquet);
   ```

5. Load data using Redshift Spectrum into Redshift internal table

     ```
     insert into klb_rs.sales_consolidate select * from klb_spectrum.sales_consolidate_parquet;
     ```

6. Observe the runtime of data load
     - Observe the performance
     - Observe the rowcount
     
          ```
          SELECT count(*) from klb_rs.sales_consolidate;
          ```
     
     - Observe the total cost of data loading process based on data size

          ```
          SELECT s3_scanned_bytes,*
          FROM SVL_S3QUERY_SUMMARY
          ```

## 5. Load to Redshift using Redshift COPY command (Option 2)

The other option is to use COPY command to load data to Redshift natively. This method **does not incur separate charges** other than than the Redshift cluster itself.

1. Create empty table based on the previous table that we have created.

     ```
     create table klb_rs.sales_consolidate_copy (like klb_spectrum.sales_consolidate_parquet);
     ```

2. Execute COPY command

     ```
     COPY klb_rs.sales_consolidate_copy
     FROM 's3://[BUCKETNAME]/data/raw/sales_consolidate_parquet/'
     iam_role 'arn:aws:iam::[YOUR-ACCOUNT-NUMBER]:role/myspectrum_role'
     FORMAT PARQUET FILLRECORD;
     ```
     
3. Observe the runtime of data load
     - Observe the performance
     - Observe the cost implication
     - Observe the rowcount
     
     ```
     SELECT count(*) from klb_rs.sales_consolidate_copy;
     ```

## 6. Run Queries in Redshift

### 6.1 Create master tables
1. Open **Redshift Query Editor v2**
2. Run the following queries:
3. Create master tables
     - Create master supplier table
          ```
          create table klb_rs.master_supplier as
          select distinct sup_name,namasup,group_supplier from klb_rs.sales_consolidate;
          ```
     - Create master product table
          ```
          create table klb_rs.master_product as
          select distinct kodeprod,klasprod,prodname,namaklasprod from klb_rs.sales_consolidate;
          ```
     - Create master branch table
          ```
          create table klb_rs.master_branch as
          select distinct kodecab,namacab from klb_rs.sales_consolidate;
          ```


### 6.2 Run Analytics Query and Optimize your table

The default redshift table has AUTO distribution style, AUTO sort key, and AUTO compression. These configurations are working for simple use cases but as your data grow you may want to specify specific values.

- SORTKEY is used to accelerate query by filtering storage blocks which does not contain your data. Sort key typically uses time-series column and other column that is frequently used as query filter.
- DISTKEY is how the data are distributed across multiple slices and nodes in your cluster. Distribution style has multiple options: `EVEN`, `KEY`, and `ALL` depending on use cases. 
     - KEY distribution style typically uses a column that frequently joined with another table with similar DISTKEY to colocate the data in the same nodes. However the DISTKEY should have high enough cardinality to avoid data skew.
     - ALL distribution style will have a data copied in each node, typically used by medium to small reference master tables.

1. Create a table with specified SORTKEY
     - In this example we are using `tgldokjdi` and `sup_name` as sortkey

     ```
     create table klb_rs.sales_consolidate_sorted
     sortkey (tgldokjdi,sup_name)
     as select * from klb_rs.sales_consolidate
     ```

2. Query into the new table (sorted). Compare the performance against the previous table. 

     Note: **First run of a query includes compilation process. Compare the performance using the second and subsequent runs only.**

     Query runtime is visible in the **lower right corner** of Redshift Query Editor v2.

     ```
     -- Disable cache for this process
     SET enable_result_cache_for_session TO OFF;

     -- Unsorted table
     select tgldokjdi, sup_name, city, count(*) cnt, sum(tot1) sum_tot1, sum(banyak) sum_banyak 
     from "klb_rs"."sales_consolidate" where tgldokjdi>'02/OCT/21 00:00:00' 
     group by tgldokjdi, sup_name, city order by tgldokjdi, sup_name, city;

     -- Sorted table
     select tgldokjdi, sup_name, city, count(*) cnt, sum(tot1) sum_tot1, sum(banyak) sum_banyak 
     from "klb_rs"."sales_consolidate_sorted" where tgldokjdi>'02/OCT/21 00:00:00' 
     group by tgldokjdi, sup_name, city order by tgldokjdi, sup_name, city;
     ```
3. Once you're done, reset the cache back on again.
     ```
     -- Disable cache for this process
     SET enable_result_cache_for_session TO ON;
     ```


## 7. Load to Redshift using Glue Studio (Option 3)

In this option we are going to explore AWS Glue as data ingestion tools to load data into Redshift

### 7.1 Setup S3 Gateway Endpoint in VPC

In this step, we will create S3 Gateway Endpoint so that Redshift cluster can communicate with S3 using its private IP.

1. Go to: **AWS VPC Console** > Click **Endpoints**
2. Click **Create endpoint**
     - Name tag - optional: `RedshiftS3EP`
     - Select AWS Services under Service category (which is the default selection)
     - Under Service name search box, search for `s3` and hit enter/return.
     - `com.amazonaws.[region].s3` should come up as search result. Select this option with type as `Gateway`.
     - Under VPC, choose non-default VPC. This is the same VPC which was used for configuring redshift cluster.
          - If you have more than VPC listed in the drop down list, double check Redshift VPC to avoid any confusion. Do the following:
          - Go to: Redshift Console  
          - Click on Redshift cluster name
          - Click on Properties tab.
          - Scroll down and check Network and security section for VPC name.
     - Once you have double checked VPC id, move to configuring Route tables section.
     - Select the listed route tables (**check both route table**)
3. Click **Create endpoint**. It should take a couple of seconds to provision this. Once this is ready, you should see Status as Available against the newly created S3 endpoint.

### 7.2 Setup Glue Connection

1. Open **AWS Glue** page
2. Open `Data Connections` page, create connection in Glue by clicking **Create Connection**
     - Enter connection name `redshift-cluster-connection-dev`
     - Choose connection type `Redshift`
     - Choose Database instances `consumercluster-xxxxxxx`
     - Database name `dev`
     - Username `awsuser`
     - Password `Awsuser123`
     - Click **Create Connection**

### 7.3 Create Glue Job for loading into Redshift
1. Open **ETL Jobs** page
2. Choose **Visual with a source and target**
3. Choose Source `AWS Glue Data Catalog` and Target `Amazon Redshift`
4. Click **Create** on upper right
5. Glue Studio Editor page is opened.
     - Choose Data source 
          - Database: `klb_db`
          - Table: `Sales_consolidate`
     - Add new transformation by selecting the + button if required
          - For example, add `Select Fields` to choose specific column
     - Choose Data target
          - Choose **Direct data connection**
          - Select Redshift connection `redshift-cluster-connection-dev`
          - Select Schema `klb_rs`
          - Select new table name `sales_consolidate_glue` (if not created yet, we can type it)
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
     
### 7.3 Create Glue Job for loading into Redshift for incremental load of master tables

Glue supports incremental load to **master table** based on the specific key. Follow the instructions below to create a job that will load into a master table.

1. Open **ETL Jobs** page
2. Choose **Visual with a source and target**
3. Choose Source `AWS Glue Data Catalog` and Target `Amazon Redshift`
4. Click **Create** on upper right
5. Glue Studio Editor page is opened.
     - Choose Data source 
          - Database: `klb_db`
          - Table: `Sales_consolidate`
     - **Add new transformation by selecting the + button if required**
          - Add `Select Fields` to choose specific column for this master table. 
          - For example, `kodecab`, `namacab`
          - Add 'Drop Duplicate' to get distinct values only
     - Choose Data target
          - Choose **Direct data connection**
          - Select Redshift connection `redshift-cluster-connection-dev`
          - Select Schema `klb_rs`
          - Select table name `master_branch`
     - **Ensure to UPDATE existing keys and INSERT new rows**
          - Select `MERGE into target table`
          - Select `kodecab` as `matching keys`
          - When matched: `Update table with the data from source`
          - When not matched: `Insert source data as a new row into table`
     - Save job as `klb_load_master_branch`
     - Click Job Details
          - Choose IAM Role `AnalyticsworkshopGlueRole`
          - Enable `Automatically scale the number of workers`
     - Click **Save**
     - Click **Run**
 6. The same method is also applicable for other master tables


## 8. Fine-grained Access Control in Redshift
1. Open **Redshift Query Editor v2**

2. Configure users
     ```
     CREATE ROLE supplier_1;
     CREATE ROLE supplier_2;
     
     CREATE USER alice WITH PASSWORD 'Awsuser123';
     CREATE USER bob WITH PASSWORD 'Awsuser123';
     
     GRANT ROLE supplier_1 to alice;
     GRANT ROLE supplier_2 to bob;
     
     GRANT usage on schema klb_rs TO ROLE supplier_1;
     GRANT usage on schema klb_rs TO ROLE supplier_2;
     ```
3. Setup Column-level access control

     - Create GRANT for different columns for each ROLE:
          ```
          GRANT select(tgldokjdi, sup_name,dept_name,group_kalbe,namacab) 
          ON TABLE klb_rs.sales_consolidate TO ROLE supplier_1;
          
          GRANT select(nodokjdi,city,tgldokjdi, sup_name,dept_name) 
          ON TABLE klb_rs.sales_consolidate TO ROLE supplier_2;
          ```
     - Impersonate different users `alice` and `bob` who have different **Column-Level Security**
          ```
          SET SESSION AUTHORIZATION alice; 
          select * from klb_rs.sales_consolidate;
          
          SET SESSION AUTHORIZATION bob; 
          select * from klb_rs.sales_consolidate;
          ```
     - Once you're done, reset the session id so that you are back to the original user `awsuser`.
     
          ```
          RESET SESSION AUTHORIZATION;
          ```
          
4. Setup Row-based access control
     - Create row-level security (RLS):
          ```
          CREATE RLS POLICY policy_supplier_1
          WITH ( sup_name varchar(16383) )
          USING ( sup_name = 'PT. SANGHIANG PERKASA' );

          CREATE RLS POLICY policy_supplier_2
          WITH ( sup_name varchar(16383) )
          USING ( sup_name = 'HEXPHARM' );

          ATTACH RLS POLICY policy_supplier_1 ON klb_rs.sales_consolidate TO ROLE supplier_1;
          ATTACH RLS POLICY policy_supplier_2 ON klb_rs.sales_consolidate TO ROLE supplier_2;

          ALTER TABLE klb_rs.sales_consolidate row level security on;
          ```
     - Impersonate different users `alice` and `bob` who have different **Row-Level Security**
          ```
          SET SESSION AUTHORIZATION alice;     
          SELECT * from klb_rs.sales_consolidate;
          select count(*), sup_name from klb_rs.sales_consolidate group by sup_name;

          SET SESSION AUTHORIZATION bob;     
          SELECT * from klb_rs.sales_consolidate;
          select count(*), sup_name from klb_rs.sales_consolidate group by sup_name;
          ```
     
5. Once you're done, reset the session id so that you are back to the original user `awsuser`.
     
     ```
     RESET SESSION AUTHORIZATION;
     ALTER TABLE klb_rs.sales_consolidate ROW LEVEL SECURITY OFF;

     ```

## 9. Orchestrate Redshift Queries using Step Function

AWS Step Functions is a serverless orchestration service that lets you integrate AWS services to build business-critical applications. Step Functions is based on state machines and tasks. A state machine is a workflow. A task is a state in a workflow that represents a single unit of work that another AWS service performs. Each step in a workflow is a state.

Step Function is very low cost with 4000 state transition free tier per month, and for each 1000 state transitions only costs $0.025.


### 9.1 Create Step Function to run Redshift Query
1. Open **Step Function** console page
2. Navigate on the left side to open **State Machines**
3. Click on **Create state machine**
     -  Step 1: Choose `Standard` type
     -  Step 2: Click on `Import/Export` then choose **Import definitions...**. 
          -   Upload the following file: [redshift-run-query-byparam.asl.json](https://raw.githubusercontent.com/nicoanadt/klb-workshop/main/step-functions/redshift-run-query-byparam.asl.json)
     -  Step 3: Click **Next**
     -  Step 4: Specify settings
          - State machine name: `redshift-run-query-byparam`
          - Permissions: `Create a new role`
          - Click **Create state machine**
     - Click on the IAM role that just been created (`Edit role in IAM`)
4. Open **IAM Console**, click **Role**, and open role `StepFunctions-redshift-run-query-byparam-role-xxxxx`
     - Click **Add Permissions** > **Attach policies**
     - Select `AmazonRedshiftDataFullAccess` , `AmazonRedshiftFullAccess` , `AWSGlueServiceRole` , `CloudWatchEventsFullAccess`, and `AWSStepFunctionsFullAccess`
     - Click **Add Permissions**
5. Note down the ARN of the step function for referral by the next process 
6. Back in Step Functions console page, observe what you have just create:
     - **Definition** of the step functions
     - What is being done in this workflow. It will trigger the redshift query using Redshift Data API, and check the execution status until it completes.

### 9.3 Create Step Function to orchestrate the end-to-end workflow

For this step we will create a workflow that will call the previous workflow as nested workflow.

1. Open Step Function console page
2. Navigate on the left side to open **State Machines**
3. Click on **Create state machine**
     -  Step 1: Choose `Standard` type
     -  Step 2: Click on `Import/Export` then choose **Import definitions...**. 
          -   Upload the following file: [redshift-run-ELT-workflow](https://raw.githubusercontent.com/nicoanadt/klb-workshop/main/step-functions/redshift-run-ELT-workflow.asl.json)
          -   Update the configuration in `Generate Execution ID` state:               
               - Click on `Output`, Add the following parameters based on the information of **your** Redshift cluster
               ```
               {
                 "rs_database": "dev",
                 "rs_user": "awsuser",
                 "rs_cluster_ident": "consumercluster-xxxxxxxx"
               }
               ```
          - Update state machine ARN for each of repopulate jobs.
               - StateMachineArn: `arn:aws:states:ap-southeast-1:[YOUR-ACCOUNT-NUMBER]:stateMachine:redshift-run-query-byparam`
     -  Step 3: Click **Next**
     -  Step 4: Specify settings
          - State machine name: `redshift-run-ETL-workflow`
          - Permissions: `Choose an existing role` : `StepFunctions-redshift-run-query-byparam-xxxxxxx`
          - Click **Create state machine**
5. Observe what you have just created, we pass the query that we want to run to `redshift-run-query-byparam` workflow for execution.
     - Observe the `rs_sql_statement` that is being executed. For example, the following `Repopulate Branch Reference Table` state will execute the following master table update:
     ```
     begin transaction; 
     create temp table master_branch_tmp as select distinct kodecab,namacab from klb_rs.sales_consolidate; 
     delete from klb_rs.master_branch using master_branch_tmp where master_branch_tmp.kodecab = master_branch.kodecab; 
     insert into klb_rs.master_branch select * from master_branch_tmp; 
     end transaction;
     ```
6. Click `Start Execution` to trigger the workflow
7. Observe whether it is successfull or not.

## 10. Orchestrate Glue jobs using Step Function

It is also possible to use Step Function to orchestrate Glue jobs! You can try it on your own.

1. Choose Glue `StartJobRun` component in Step Function
2. Include the Glue `JobName` as Input component of the state
     ```
     {
       "JobName": "myGlueJobName"
     }
     ```
3. Check the box  `Wait for the task to complete` to wait for each job completion in the workflow.




