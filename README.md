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

5. Copy all partitions inside `s3://kalbe-workshop-test-130835040051-apse1/data/sales_consolidate/` to your bucket folder

    `s3:/[BUCKETNAME]/data/raw/sales_consolidate/`
