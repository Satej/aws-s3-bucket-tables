# aws-s3-bucket-tables

```bash
aws s3tables create-table-bucket \
    --region us-east-1 \
    --name a-unique-bucket-name
```

Output:
```bash
{
    "arn": "arn:aws:s3tables:us-east-2:851725604111:bucket/a-unique-bucket-name"
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

aws iam create-instance-profile --instance-profile-name EMR_EC2_DefaultRole
aws iam add-role-to-instance-profile \
    --instance-profile-name EMR_EC2_DefaultRole \
    --role-name EMR_EC2_DefaultRole


# Create a EMR Cluster and Spark session.
aws emr create-cluster --release-label emr-7.5.0 \
--applications Name=Spark \
--configurations file://configurations.json \
--region us-east-1 \
--name My_Spark_Iceberg_Cluster \
--log-uri s3://a-unique-bucket-name/ \
--instance-type m5.xlarge \
--instance-count 2 \
--service-role EMR_DefaultRole_V2 \
--ec2-attributes \
InstanceProfile=EMR_EC2_DefaultRole,SubnetId=subnet-1234567890abcdef0
```

Output
```bash
{
    "ClusterId": "j-WS6YLQOL8EAK",
    "ClusterArn": "arn:aws:elasticmapreduce:us-east-1:851725604111:cluster/j-WS6YLQOL8EAK"
}
```
