# alluxio-on-emr
Launch Alluxio Data Orchestration on EMR with Presto, Spark, Hive and JupyterLab

## Background

Alluxio Data Orchestration enables data engineers to allow data consumers to access data anyware in the enterprise. Whether the data is in an on-prem storage environment like Hadoop or S3 compatible storage, or in a cloud- based datalake, Alluxio's unified-namespace feature allows Presto, Impala, Drill, Dremio, Spark and Hive users to access remote data without knowing that the data is remote. And Alluxio's memory caching allows those users to access the data at "local data" query speeds.

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

## Step 2. Create a public ssh key and AWS keypair

Generate a private and public ssh key using a command like this:

     ssh-keygen -t rsa -N '' -f ./alluxio_emr_id_rsa <<< y

Import the newly generated ssh key and create a keypair in your AWS environment. Use this command:

     aws --region us-east-1 ec2 \
        import-key-pair \
        --key-name "alluxio-emr-keypair" \
        --public-key-material fileb://alluxio_emr_id_rsa.pub

## Step 3. Create the S3 bucket for this EMR

Use the following commands to create the S3 bucket:

     aws --region us-east-1 s3api create-bucket --bucket alluxio-emr-bucket \
          --region us-east-1 \
          --acl private 

## Step 4. Launch the Alluxio EMR cluster

Use the following command to launch the Alluxio EMR cluster.  Modify the following arguments as needed:

     --name "my-alluxio-emr-cluster-1"
     --tags  "Name=my-alluxio-emr-cluster-1"
     --instance-type r4.4xlarge
     --instance-count 5

Note that the alluxio-emr.sh script does NOT configure the Alluxio Catalog Service with the Presto plugin. Running the Alluxio Catalog Service is not supported in the newer releases of Presto on EMR.

Use the following create-cluster command:

     aws --region us-east-1 emr create-cluster \
	        --name "my-alluxio-emr-cluster-1" \
	        --tags "Name=my-alluxio-emr-cluster-1" \
	        --release-label emr-6.7.0 \
	        --instance-count 5 \
	        --instance-type r4.4xlarge \
	        --applications Name=Presto Name=Hive Name=Spark Name=JupyterHub \
	        --configurations '[{"Classification":"hive-site","Properties":{"hive.metastore.client.factory.class":"com.amazonaws.glue.catalog.metastore.AWSGlueDataCatalogHiveClientFactory"}},{"Classification":"presto-connector-hive","Properties":{"hive.metastore":"glue","hive.s3-file-system-type":"PRESTO"}},{"Classification":"hadoop-env","Configurations":[{"Classification":"export","Properties":{"HADOOP_CLASSPATH":"/opt/alluxio/client/alluxio-client.jar:${HADOOP_CLASSPATH}"}}],"Properties":{}}]' \
	        --ec2-attributes KeyName=alluxio-emr-keypair \
	        --ebs-root-volume-size 30 \
	        --log-uri s3://alluxio-emr-bucket/emr-cluster-logs \
	        --bootstrap-actions 'Path=s3://greg-palmer-alluxio-public-bucket/alluxio-on-emr/alluxio-emr.sh,Args=[s3://alluxio-emr-bucket,-d,"https://downloads.alluxio.io/downloads/files/2.8.1/alluxio-2.8.1-bin.tar.gz",-p,"alluxio.user.block.size.bytes.default=122M^alluxio.user.file.writetype.default=CACHE_THROUGH^alluxio.master.web.port=19995^alluxio.master.rpc.port=19994",-s,"^"]'

## Step 5. Get the public IP address of the EMR master node

Get the public IP address of the EMR master node by using the AWS EC2 instance console, or run the following CLI command:

     aws ec2 describe-instances \
          --filters Name=tag:"Name",Values="my-alluxio-emr-cluster-1"  \
                    Name=tag:"aws:elasticmapreduce:instance-group-role",Values="MASTER" \
                    Name=instance-state-name,Values="running" \
          --query  'Reservations[*].Instances[*].[PublicIpAddress]' \
          --output  text
          
# Usage Instructions

## Step 6. Open an SSH session into the EMR master node 

Use the following command to open a secure shell session into the EMR master node. Use the public IP address displayed in Step 5 above.

     ssh -i ./alluxio_emr_id_rsa hadoop@<master node public ip>

## Step 7. Create a test Hive table pointing to the Alluxio file system

Alluxio has been configured to use the S3 bucket specified in the EMR `create-cluster` command as the root "under file system" or UFS. Create some test data in that S3 bucket to be used by Alluxio and query engines accessing Alluxio.

Using the Alluxio CLI, create a new directory in the S3 bucket and import some test data from the TPC-DS data set.

     alluxio fs mkdir /data/tpcds

     s3-dist-cp --src s3a://autobots-tpcds-pregenerated-data/spark/unpart_sf100_10k/store_sales/ --dest alluxio:///data/tpcds/store_sales

     s3-dist-cp --src s3a://autobots-tpcds-pregenerated-data/spark/unpart_sf100_10k/item/ --dest alluxio:///data/tpcds/item

Confirm that the data files were copied to the Alluxio under file system:

     alluxio fs ls -R /data/tpcds/ | more

Create a Hive table that points to the Alluxio (S3) data set. After downloading the hive SQL script, you will notice that it does not point to HDFS or S3 directly, instead it references the Alluxio file system like this:

     CREATE EXTERNAL TABLE IF NOT EXISTS store_sales (
      ...
     STORED AS PARQUET LOCATION 'alluxio:///data/tpcds/store_sales/'

This is possible because Hive has been configured to integrate with Alluxio just like Presto and Spark have been.

Run these commands:

     wget https://raw.githubusercontent.com/gregpalmr/alluxio-on-emr/main/hive/create-hive-tables.sql

     hive -f create-hive-tables.sql

Make sure Hive can access the TPC-DS dataset in Alluxio:

     hive --database alluxio_db -e "SELECT * FROM store_sales LIMIT 10;"

## Step 8. Query the TPC-DS data using the Presto query engine and Alluxio

First, make a note that the TPC-DS data set in S3 is not yet cached by the Alluxio worker nodes. Run the "alluxio fs admin" command and notice that the "Used Capacity" is shown at 0B:

     alluxio fsadmin report

Use the Presto CLI to run the TPC-DS query 44 that gets the top 10 stores with the highest sales levels. This Presto query will access the Alluxio TPC-DS data stored in the S3 UFS. Use this command:

    wget https://raw.githubusercontent.com/gregpalmr/alluxio-on-emr/main/hive/q44.sql

    time presto-cli --catalog hive --schema alluxio_db -f q44.sql 

Note that it took about 48 seconds to run the TPC-DS 44 query.

Now run the "alluxio fsadmin report" command again and notice that Alluxio has cached the store_sales data set on the Alluxio worker nodes. The "Used Capacity" is now showing around 38 GB:

     alluxio fsadmin report

The "Used Capacity" value has increased because Alluxio automatically cached the store_sales data in S3 when the Presto query first requested the data. Users can also pre-load data into the Alluxio cache using the "alluxio fs load" command.

Now re-run the Presto query and see if the query takes less time:

    time presto-cli --catalog hive --schema alluxio_db -f q44.sql 

The query should be about 30% faster after Alluxio cached the S3 data locally. This cached data is not limited to Presto usage, any query against Alluxio can benefit from the cached data, including Spark, Hive, JupyterLab, Impala, Dremio and other users.

Finally, unload the data from the Alluxio cache using the commands:

     alluxio fs free /data

     alluxio fsadmin report

# Destroy EMR Cluster Instructions

## Step 9. Destroy the EMR cluster and supporting AWS objects.

Use the following command to get the EMR cluster ID:

     emr_cluster_id=$(aws emr list-clusters --active | grep my-alluxio-emr-cluster-1 | cut -f 3)

Use the following command to shutdown the Alluxio EMR cluster:

     aws emr terminate-clusters --cluster ${emr_cluster_id}

Use the following command to remove the AWS key pair used with the EMR cluster:

     aws --region us-east-1 ec2 delete-key-pair --key-name alluxio-emr-keypair

Use the following command to remove the S3 bucket used with the EMR cluster:

     aws --region us-east-1 s3api delete-bucket  --bucket alluxio-emr-bucket


