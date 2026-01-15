# commands.md — EMR CLI / SSH / Hive Queries

This file records the key commands used in this project to connect to an Amazon EMR cluster and run HiveQL queries to process CloudFront logs stored in S3.

---

## Environment

- **Region**: `us-east-1`
- **EMR Release**: `emr-5.36.1` (lab reference) / compatible on newer releases
- **Log sample input (S3)**: `s3://us-east-1.elasticmapreduce.samples`
- **Hive script (S3)**: `s3://us-east-1.elasticmapreduce.samples/cloudfront/code/Hive_CloudFront.q`
- **Output (S3)**: `s3://hadoop3023/os_requests/`

> ⚠️ Do NOT commit AWS credentials or `.pem` keypairs into this repo.

---

## 0) Prerequisites

- AWS CLI installed
- IAM permissions to access **EMR + EC2 + S3**
- `jq` installed (for parsing JSON in CLI)
- EMR EC2 keypair (`.pem`) available locally

Example (key permission):
```bash
chmod 400 ~/EMRKey-lab.pem
```

1) List EMR clusters (find the correct one)
bash
Copy code
aws emr list-clusters --active --region us-east-1
If your cluster is already terminated (not active), use:

bash
Copy code
aws emr list-clusters --cluster-states TERMINATED --region us-east-1
2) Get EMR Cluster ID (store into variable)
Option A (pick the first cluster in the list):

bash
Copy code
export ID=$(aws emr list-clusters --active --region us-east-1 | jq -r '.Clusters[0].Id')
echo $ID
Option B (find by cluster name — recommended)
Replace <CLUSTER_NAME> with your EMR cluster name:

bash
Copy code
export ID=$(aws emr list-clusters --active --region us-east-1 \
  | jq -r '.Clusters[] | select(.Name=="<CLUSTER_NAME>") | .Id')
echo $ID
3) Get EMR master node Public DNS
bash
Copy code
export HOST=$(aws emr describe-cluster --cluster-id $ID --region us-east-1 \
  | jq -r '.Cluster.MasterPublicDnsName')
echo $HOST
4) SSH into EMR master node
bash
Copy code
ssh -i ~/EMRKey-lab.pem hadoop@$HOST
First time login may show:

bash
Copy code
Are you sure you want to continue connecting (yes/no)?
Type:

bash
Copy code
yes
5) Open Hive on EMR
After SSH into master node:

bash
Copy code
hive
You should see a prompt like:

sql
Copy code
hive>
6) Verify Hive tables (optional)
sql
Copy code
SHOW DATABASES;
Switch to default DB:

sql
Copy code
USE default;
List tables:

sql
Copy code
SHOW TABLES;
Confirm cloudfront_logs exists:

sql
Copy code
DESCRIBE cloudfront_logs;
7) Run the HiveQL query (OS request counts)
This is the main query you executed on the EMR cluster:

sql
Copy code
SELECT
  os,
  COUNT(*) AS count
FROM cloudfront_logs
WHERE dateobject BETWEEN '2014-07-05' AND '2014-08-05'
GROUP BY os;
Expected output: aggregated OS counts such as:

Android

Windows

MacOS / OS X

Linux

iOS

8) Write query output back to S3 (recommended)
To save results into S3 output folder:

sql
Copy code
INSERT OVERWRITE DIRECTORY 's3://hadoop3023/os_requests/'
SELECT
  os,
  COUNT(*) AS count
FROM cloudfront_logs
WHERE dateobject BETWEEN '2014-07-05' AND '2014-08-05'
GROUP BY os;
This produces output part files in S3 (example):

000000_0

000001_0

9) Exit Hive and SSH session
Exit Hive:

sql
Copy code
exit;
Exit EMR SSH:

bash
Copy code
exit
10) Download results from S3 (local validation)
List output files:

bash
Copy code
aws s3 ls s3://hadoop3023/os_requests/ --region us-east-1
Download all outputs into a local folder:

bash
Copy code
mkdir -p output
aws s3 cp s3://hadoop3023/os_requests/ output/ --recursive --region us-east-1
View file contents:

bash
Copy code
cat output/000000_0
cat output/000001_0
Troubleshooting
A) SSH Permission denied (publickey)
Key permission may be wrong → run:

bash
Copy code
chmod 400 ~/EMRKey-lab.pem
Ensure you selected the correct keypair during cluster creation.

B) Hive table not found: cloudfront_logs
You likely have not executed the Hive script yet.

Option 1: run step through EMR console
Option 2: run Hive script manually:

bash
Copy code
hive -f /home/hadoop/Hive_CloudFront.q
(If you don’t have the script locally, download it from S3 first.)

C) No output files in S3
Check:

Your INSERT OVERWRITE DIRECTORY path is correct

EMR role has permission to write to the bucket

Look at EMR step logs (stderr, stdout)

References
EMR UI screenshots: see /images/

Output samples: see /output/ (if included in repo)

