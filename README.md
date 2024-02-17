# alluxio-on-emr
Launch Alluxio Data Orchestration on EMR with Trino, Spark, Hive and JupyterLab

## Background

Alluxio Data Orchestration enables data engineers to allow data consumers to access data anyware in the enterprise. Whether the data is in an on-prem storage environment like Hadoop or S3 compatible storage, or in a cloud-based datalake, Alluxio's unified-namespace feature allows Trino, Impala, Drill, Dremio, Spark and Hive users to access remote data without knowing that the data is remote. And Alluxio's memory caching allows those users to access the data at "local data" query speeds.

This git repo contains instructions and artifacts for launching Alluxio data orchestration on an EMR cluster.

## Pre-requesites

To use the commands outlined in the repo, you will need the following:

- The git CLI installed
- The AWS CLI installed
- Your AWS credentials defined defined in the `~/.aws/credentials`, like this:

     - [default]
     - aws_access_key_id=[AWS ACCESS KEY]
     - aws_secret_access_key=[AWS SECRET KEY]

- You also need IAM role membership and permissions to create the following objects:
     - AWS Key Pairs
     - AWS S3 Buckets (or already have one available)
     - EMR clusters
     - EC2 instance types as specfied in the create-cluster command

# Setup Instructions

## Step 1. Clone this git repo

     git clone https://github.com/gregpalmr/alluxio-on-emr

     cd alluxio-on-emr

## Step 2. Create a public ssh key and AWS keypair

Generate a private and public ssh key using a command like this:

     ssh-keygen -t rsa -N '' -f ./alluxio_emr_id_rsa <<< y

Import the newly generated ssh key and create a keypair in your AWS environment. Use this command:

     aws --region us-east-1 ec2 \
        import-key-pair \
        --key-name "my-alluxio-emr-keypair" \
        --public-key-material fileb://alluxio_emr_id_rsa.pub

## Step 3. Create the S3 bucket for this EMR

Use the following commands to create the S3 bucket:

     aws --region us-east-1 s3api create-bucket --bucket alluxio-emr-bucket \
          --region us-east-1 \
          --acl private 

## Step 4. Launch the Alluxio EMR cluster

Use the following command to launch the Alluxio EMR cluster.  Modify the following arguments as needed:

     ...
     --name "my-alluxio-emr-cluster-1"
     --tags  "Name=my-alluxio-emr-cluster-1"
     --ec2-attributes KeyName=my-alluxio-emr-keypair
     --instance-type r4.4xlarge
     --instance-count 5                        (must be at least 2)
     ...

And modify the S3 bucket name as the first argument to the alluxio-enterprise-emr-bootstrap.sh script. Like this:

     ... alluxio-enterprise-emr-bootstrap.sh,Args=[s3://alluxio-emr-bucket ...

NOTE: The alluxio-enterprise-emr-bootstrap.sh script does NOT configure the Alluxio Catalog Service with the Trino plugin. Running the Alluxio Catalog Service is not supported in the newer releases of Trino on EMR. The Enterprise Edition of Alluxio provides the Transparent URI feature which allows the use of unmodified Hive table definitions to be redirected to Alluxio. See below.

Use the following create-cluster command:

aws emr create-cluster \
        --region us-east-1 \
        --name "my-alluxio-enterprise-emr-cluster-1" \
        --tags "Name=my-alluxio-enterprise-emr-cluster-1" \
        --ec2-attributes KeyName=my-alluxio-emr-keypair \
        --instance-count 5 \
        --instance-type r4.4xlarge \
        --release-label emr-7.0.0 \
        --applications Name=Trino Name=Hive Name=Spark Name=Hadoop \
        --configurations '[ { "Classification": "hive-site", "Properties": { "hive.metastore.client.factory.class": "com.amazonaws.glue.catalog.metastore.AWSGlueDataCatalogHiveClientFactory", "fs.s3.impl": "alluxio.hadoop.ShimFileSystem", "fs.AbstractFileSystem.s3.impl": "alluxio.hadoop.AlluxioShimFileSystem", "fs.s3a.impl": "alluxio.hadoop.ShimFileSystem", "fs.AbstractFileSystem.s3a.impl": "alluxio.hadoop.AlluxioShimFileSystem" } }, { "Classification": "core-site", "Properties": { "fs.s3.impl": "alluxio.hadoop.ShimFileSystem", "fs.AbstractFileSystem.s3.impl": "alluxio.hadoop.AlluxioShimFileSystem", "fs.s3a.impl": "alluxio.hadoop.ShimFileSystem", "fs.AbstractFileSystem.s3a.impl": "alluxio.hadoop.AlluxioShimFileSystem", "alluxio.user.shimfs.allow.list": "s3://alluxio-emr-bucket/", "fs.s3a.server-side-encryption-algorithm": "AES256" } }, { "Classification": "trino-connector-hive", "Properties": { "hive.metastore": "glue", "hive.s3-file-system-type": "HADOOP_DEFAULT" } }, { "Classification": "trino-connector-iceberg", "Properties": { "hive.s3-file-system-type": "HADOOP_DEFAULT" } }, { "Classification": "hadoop-env", "Configurations": [ { "Classification": "export", "Properties": { "HADOOP_CLASSPATH": "/opt/alluxio/client/alluxio-client.jar:${HADOOP_CLASSPATH}" } } ], "Properties": {} } ]' \
        --ebs-root-volume-size 30 \
        --log-uri s3://alluxio-emr-bucket/emr-cluster-logs \
        --bootstrap-actions 'Path=s3://greg-palmer-alluxio-public-bucket/alluxio-on-emr/alluxio-enterprise-emr-bootstrap.sh,Args=[s3://alluxio-emr-bucket,-d,"https://downloads.alluxio.io/protected/files/alluxio-enterprise-trial.tar.gz",-p,"alluxio.user.block.size.bytes.default=16M^alluxio.user.file.writetype.default=CACHE_THROUGH^alluxio.master.shimfs.auto.mount.enabled=true^alluxio.master.shimfs.auto.mount.readonly=false^alluxio.user.shimfs.allow.list=s3://alluxio-emr-bucket2",-s,"^"]'

## Step 5. Get the public IP address of the EMR master node

Get the public IP address of the EMR master node by using the AWS EC2 instance console, or run the following CLI command:

     aws ec2 describe-instances \
          --filters Name=tag:"Name",Values="my-alluxio-enterprise-emr-cluster-1"  \
                    Name=tag:"aws:elasticmapreduce:instance-group-role",Values="MASTER" \
                    Name=instance-state-name,Values="running" \
          --query  'Reservations[*].Instances[*].[PublicIpAddress]' \
          --output  text
          
# Usage Instructions

## Step 6. Open an SSH session into the EMR master node 

Use the following command to open a secure shell session into the EMR master node. Use the public IP address displayed in Step 5 above.

     ssh -i ./alluxio_emr_id_rsa hadoop@<master node public ip>

## Step 7. Access Alluxio and Trino Web UIs

Note: You may have to modify the AWS Security Group that is associated with your EMR master node to allow access to the Alluxio and Trino Web UIs. Add ingress rules that allow access to ports 19995 and 8889 to your workstation or laptop computer.

Using the public IP address displayed in Step 5 above, access the Alluxio Web UI at:

     http://<public ip address>:19995

This console will show you summary and detailed information about the Alluxio cluster, including master node information, worker node information and performance metrics.

Access the Trino Web UI at:

     http://<public ip address>:8889

## Step 8. Load TPC-DS data into Alluxio and the S3 under file system (UFS)

Alluxio has been configured to use the S3 bucket specified in the EMR `create-cluster` command as the root "under file system" or UFS. Create some test data in that S3 bucket to be used by Alluxio and query engines accessing Alluxio.

Using the Alluxio CLI, create a new directory in the S3 bucket and import some test data from the TPC-DS data set.

     alluxio fs mkdir /data/tpcds

     s3-dist-cp --src s3a://autobots-tpcds-pregenerated-data/spark/unpart_sf100_10k/store_sales/ --dest alluxio:///data/tpcds/store_sales

     s3-dist-cp --src s3a://autobots-tpcds-pregenerated-data/spark/unpart_sf100_10k/item/ --dest alluxio:///data/tpcds/item

Confirm that the data files were copied to the Alluxio under file system:

     alluxio fs ls -R /data/tpcds/ | more

## Step 9. Use Alluxio Transparent URI with existing Hive tables 

If you are using the Enterprise Edition of Alluxio, then you can take advantage of Alluxio's Transparent URI capability. This will allow you to use the same Hive table definitions that you currently use for direct access to your data (such as hdfs:// or s3://), but redirect the read/write requests to the Alluxio virtual file system.

To demonstrate the Transparent URI capability, create a Hive table that uses a LOCATION clause that references an S3 end-point or a hdfs end-point. It will contain a LOCATION clause like this:

     LOCATION 's3://alluxio-emr-bucket/data/tpcds/store_sales/'

Run these commands:

     wget https://raw.githubusercontent.com/gregpalmr/alluxio-on-emr/main/hive/create-hive-tables-s3.sql

     hive -f create-hive-tables-s3.sql

NOTE: If you changed the name of the S3 bucket specified in the first argument of the alluxio-enteprise-emr-bootstrap.sh script, then modify the create-hive-tables-s3.sql to use that bucket name. See "Args=[s3://alluxio-emr-bucket" in the create-cluster command in Step 4.

Make sure Hive can access the S3 dataset via Alluxio's Transparent URI capability. In this case, Hive will use the Alluxio "shim filesystem" configuration.

     hive --database tpcds_s3_db -e "SELECT * FROM store_sales LIMIT 10;"

## Step 10. Query the TPC-DS data using the Trino query engine and Alluxio Transparent URI 

First, free any cached TPC-DS data in Alluxio:

     alluxio fs free /data

Then, make a note that the TPC-DS data set in S3 is not yet cached by the Alluxio worker nodes. Run the "alluxio fs admin" command and notice that the "Used Capacity" is shown at 0B:

     alluxio fsadmin report

Use the Trino CLI to run the TPC-DS query 44 that gets the top 10 stores with the highest sales levels. This Trino query will access the Alluxio TPC-DS data stored in the S3 UFS. Use this command:

    wget https://raw.githubusercontent.com/gregpalmr/alluxio-on-emr/main/hive/q44.sql

    time trino-cli --catalog hive --schema tpcds_s3_db -f q44.sql 

Note that it took about 56 seconds to run the TPC-DS 44 query with 4 Trino and Alluxio worker nodes.

Now run the "alluxio fsadmin report" command again and notice that Alluxio has cached the store_sales data set on the Alluxio worker nodes. The "Used Capacity" is now showing around 43 GB (with 4 Alluxio worker nodes caching data):

     alluxio fsadmin report

The "Used Capacity" value has increased because Alluxio automatically cached the store_sales data in S3 when the Trino query first requested the data. Users can also pre-load data into the Alluxio cache using the "alluxio fs load" command.

Now re-run the Trino query and see if the query takes less time:

    time trino-cli --catalog hive --schema tpcds_s3_db -f q44.sql 

The query should be about 42% faster and taking only 32 seconds to run, after Alluxio cached the S3 data locally. This cached data is not limited to Trino usage, any query against Alluxio can benefit from the cached data, including Spark, Hive, JupyterLab, Impala, Dremio and other users.

Finally, unload the data from the Alluxio cache using the commands:

     alluxio fs free /data

     alluxio fsadmin report

## Step 11. Create a test Hive table pointing to the Alluxio virtual file system end-point

If you have existing Hive table definitions in your Hive or Glue metastore, then you will have to modify your Hive table definitions to switch to a location URI that references the Alluxio end-point. For tables that are already defined, alter the location with a command like this:

     hive> ALTER TABLE <my table> SET LOCATION "alluxio://<emr master node ip address>:19994/user/hive/warehouse/<my table dir>";

To simulate this, create a Hive table that points to the Alluxio virtual file system. After downloading the hive SQL script, you will notice that it does not point to HDFS or S3 directly, instead it references the Alluxio virtual file system like this:

     CREATE EXTERNAL TABLE IF NOT EXISTS store_sales (
      ...
     LOCATION 'alluxio:///data/tpcds/store_sales/'

This is possible because Hive has been configured to integrate with Alluxio just like Trino and Spark have been.

Run these commands:

     wget https://raw.githubusercontent.com/gregpalmr/alluxio-on-emr/main/hive/create-hive-tables-alluxio.sql

     grep LOCATION create-hive-tables-alluxio.sql

     hive -f create-hive-tables-alluxio.sql

Make sure Hive can access the TPC-DS dataset in Alluxio:

     hive --database tpcds_alluxio_db -e "SELECT * FROM store_sales LIMIT 10;"

## Step 12. Query the TPC-DS data using the Trino query engine and the Alluxio virtual file system end-point

First, free any cached TPC-DS data in Alluxio:

     alluxio fs free /data

Then, make a note that the TPC-DS data set in S3 is not yet cached by the Alluxio worker nodes. Run the "alluxio fs admin" command and notice that the "Used Capacity" is shown at 0B:

     alluxio fsadmin report

Use the Trino CLI to run the TPC-DS query 44 that gets the top 10 stores with the highest sales levels. This Trino query will access the Alluxio TPC-DS data stored in the S3 UFS. Use this command:

    wget https://raw.githubusercontent.com/gregpalmr/alluxio-on-emr/main/hive/q44.sql

    time trino-cli --catalog hive --schema tpcds_alluxio_db -f q44.sql 

Note that it took about 56 seconds to run the TPC-DS 44 query with 4 Trino and Alluxio worker nodes.

Now run the "alluxio fsadmin report" command again and notice that Alluxio has cached the store_sales data set on the Alluxio worker nodes. The "Used Capacity" is now showing around 43 GB (with 4 Alluxio worker nodes caching data):

     alluxio fsadmin report

The "Used Capacity" value has increased because Alluxio automatically cached the store_sales data in S3 when the Trino query first requested the data. Users can also pre-load data into the Alluxio cache using the "alluxio fs load" command.

Now re-run the Trino query and see if the query takes less time:

    time trino-cli --catalog hive --schema tpcds_alluxio_db -f q44.sql 

The query should be about 42% faster and taking only 32 seconds to run, after Alluxio cached the S3 data locally. This cached data is not limited to Trino usage, any query against Alluxio can benefit from the cached data, including Spark, Hive, JupyterLab, Impala, Dremio and other users.

Finally, unload the data from the Alluxio cache using the commands:

     alluxio fs free /data

     alluxio fsadmin report

# Destroy EMR Cluster Instructions

## Step 13. Destroy the EMR cluster and supporting AWS objects.

Use the following command to get the EMR cluster ID:

     emr_cluster_id=$(aws emr list-clusters --active --output text | grep my-alluxio-enterprise-emr-cluster-1 | cut -f 3)

Use the following command to shutdown the Alluxio EMR cluster:

     aws emr terminate-clusters --cluster ${emr_cluster_id}

Use the following command to remove the AWS key pair used with the EMR cluster:

     aws --region us-east-1 ec2 delete-key-pair --key-name my-alluxio-emr-keypair

Use the following command to remove the S3 bucket used with the EMR cluster:

     aws --region us-east-1 s3 rm s3://alluxio-emr-bucket/ --recursive

     aws --region us-east-1 s3api delete-bucket  --bucket alluxio-emr-bucket


