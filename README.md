# aws-s3-bucket-tables

```bash
aws s3tables create-table-bucket \
    --region us-east-1 \
    --name a-unique-bucket-name
```

Output:
```bash
{
    "arn": "arn:aws:s3tables:us-east-1:851725604111:bucket/a-unique-bucket-name"
}
```

```bash
aws iam create-role \
    --role-name EMR_DefaultRole_V2 \
    --assume-role-policy-document file://<(echo '{
      "Version": "2012-10-17",
      "Statement": {
        "Effect": "Allow",
        "Principal": {
          "Service": "elasticmapreduce.amazonaws.com"
        },
        "Action": "sts:AssumeRole"
      }
    }')

aws iam create-role \
    --role-name EMR_EC2_DefaultRole \
    --assume-role-policy-document file://<(echo '{
      "Version": "2012-10-17",
      "Statement": {
        "Effect": "Allow",
        "Principal": {
          "Service": "ec2.amazonaws.com"
        },
        "Action": "sts:AssumeRole"
      }
    }')

aws iam attach-role-policy \
    --role-name EMR_EC2_DefaultRole \
    --policy-arn arn:aws:iam::aws:policy/service-role/AmazonElasticMapReduceforEC2Role
aws iam attach-role-policy \
    --role-name EMR_EC2_DefaultRole \
    --policy-arn arn:aws:iam::aws:policy/AmazonS3TablesFullAccess
aws iam attach-role-policy \
    --role-name EMR_DefaultRole_V2 \
    --policy-arn arn:aws:iam::aws:policy/service-role/AmazonElasticMapReduceRole

aws iam create-instance-profile --instance-profile-name EMR_EC2_DefaultRole
aws iam add-role-to-instance-profile \
    --instance-profile-name EMR_EC2_DefaultRole \
    --role-name EMR_EC2_DefaultRole

# Use one of the available subnets
aws ec2 describe-subnets --region us-east-1 --query "Subnets[*].{SubnetId:SubnetId, AvailabilityZone:AvailabilityZone, VpcId:VpcId, State:State}" --output table

aws ec2 create-key-pair --region us-east-1 --key-name MyEMRKey --query 'KeyMaterial' --output text > MyEMRKey.pem
chmod 400 MyEMRKey.pem

# Create a EMR Cluster and Spark session.
aws emr create-cluster \
    --release-label emr-7.5.0 \
    --applications Name=Spark \
    --configurations file://configurations.json \
    --region us-east-1 \
    --name My_Spark_Iceberg_Cluster \
    --log-uri s3://amzn-s3-demo-bucket/ \
    --instance-type m5.xlarge \
    --instance-count 2 \
    --service-role EMR_DefaultRole_V2 \
    --ec2-attributes KeyName=MyEMRKey,InstanceProfile=EMR_EC2_DefaultRole,SubnetId=subnet-0824357ac6e8b2af3
```

Output
```bash
{
    "ClusterId": "j-1L5SBP5H1NGQ0",
    "ClusterArn": "arn:aws:elasticmapreduce:us-east-1:851725604111:cluster/j-1L5SBP5H1NGQ0"
}
```
aws emr list-clusters
j-1L5SBP5H1NGQ0

Wait for cluster to be in WAITING State
aws emr describe-cluster --region us-east-1 --cluster-id j-1L5SBP5H1NGQ0  --query "Cluster.Ec2InstanceAttributes" --output text

aws ec2 authorize-security-group-ingress \
    --region us-east-1 \
    --group-id sg-03049bb3db6e39d0b \
    --protocol tcp \
    --port 22 \
    --cidr 0.0.0.0/0

aws emr describe-cluster --region us-east-1 --cluster-id j-1L5SBP5H1NGQ0 --query "Cluster.MasterPublicDnsName" --output text

ssh -i MyEMRKey.pem hadoop@<MasterPublicDnsName>

```bash
spark-shell \
--packages software.amazon.s3tables:s3-tables-catalog-for-iceberg-runtime:0.1.3 \
--conf spark.sql.catalog.s3tablesbucket=org.apache.iceberg.spark.SparkCatalog \
--conf spark.sql.catalog.s3tablesbucket.catalog-impl=software.amazon.s3tables.iceberg.S3TablesCatalog \
--conf spark.sql.catalog.s3tablesbucket.warehouse=arn:aws:s3tables:us-east-1:851725604111:bucket/a-unique-bucket-name \
--conf spark.sql.extensions=org.apache.iceberg.spark.extensions.IcebergSparkSessionExtensions

scala> spark.sql("CREATE NAMESPACE IF NOT EXISTS s3tablesbucket.example_namespace")           
res0: org.apache.spark.sql.DataFrame = []

scala> spark.sql( 
     | """ CREATE TABLE IF NOT EXISTS s3tablesbucket.example_namespace.`example_table` ( 
     |     id INT, 
     |     name STRING, 
     |     value INT 
     | ) 
     | USING iceberg """
     | )
SLF4J: Failed to load class "org.slf4j.impl.StaticLoggerBinder".
SLF4J: Defaulting to no-operation (NOP) logger implementation
SLF4J: See http://www.slf4j.org/codes.html#StaticLoggerBinder for further details.
res1: org.apache.spark.sql.DataFrame = []

spark.sql(
"""
    INSERT INTO s3tablesbucket.example_namespace.example_table 
    VALUES 
        (1, 'ABC', 100), 
        (2, 'XYZ', 200)
""")

scala> spark.sql(""" SELECT *
     | FROM s3tablesbucket.example_namespace.`example_table` """).show()
+---+----+-----+                                                                
| id|name|value|
+---+----+-----+
|  1| ABC|  100|
|  2| XYZ|  200|
```
