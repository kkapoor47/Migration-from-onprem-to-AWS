# Migrate your Existing Workload from on-prem to AWS

![Migrating](https://www.uturndata.com/wp-content/uploads/2019/07/cloudmigration5-1.png)



**We will migrate on-prem VM to AWS EC2 using some simple steps**

We will use `VM Import` to migrate existing VM. This will not only migrate but also preserve the software and settings that we had configured already in our VM
## Prerequisites
* On-prem server/ VM (In my case , I used Virtual Machine in Oracle VirtualBox)
* AWS Account with ***Administrator Priveleges*** (we'll use AWS CLI)

## Steps
1.   Export VM 
2.   Upload `.vmdk` file to S3
3.   Global Customization Variables
4.   Create Trust Policy
5.   Create IAM Role
6.   Create IAM Policy
7.   Attach IAM Policy to IAM Role
8.   Import VM Image
9.   Check the status of VM Image
10.  Import Completed
11.  Launch EC2 Machine from AMI



## 1. Export VM
Right click on your VM and click export. Export your file with proper extension either `.vmdk` or `.ovf` and choose location where you want to export in your Desktop or PC

## 2. Upload to S3
Go to location where you exported the file. Select `.vmdk` file and upload it to S3 bucket

## 3. Global Customization Variables
```
bucket_name="mig-VM-to-bucket"

<!--  Add the appropriate S3 Prefix to the VM Image -->
vm_image_name="VM-Import/vKaliOS-disk001.vmdk"
```

## 4. Create Trust Policy
Now we'll create IAM trust policy in json. Name the file as `trust-policy.json`
```

cat > "trust-policy.json" << "EOF"
{
   "Version": "2012-10-17",
   "Statement": [
      {
         "Effect": "Allow",
         "Principal": { "Service": "vmie.amazonaws.com" },
         "Action": "sts:AssumeRole",
         "Condition": {
            "StringEquals":{
               "sts:Externalid": "vmimport"
            }
         }
      }
   ]
}
EOF
```

## 5. Create IAM Role
Make sure you create role with name `vmimport` 

```
aws iam create-role --role-name vmimport --assume-role-policy-document "file://trust-policy.json"
```

## 6.  Create IAM Policy
We'll attch this policy to the above role created `vmimport`
Let's name this IAM Policy as `role-policy.json`

```
echo '{
   "Version":"2012-10-17",
   "Statement":[
      {
         "Effect":"Allow",
         "Action":[
            "s3:GetBucketLocation",
            "s3:GetObject",
            "s3:ListBucket" 
         ],
         "Resource":[
            "arn:aws:s3:::'${bucket_name}'",
            "arn:aws:s3:::'${bucket_name}'/*"
         ]
      },
      {
         "Effect":"Allow",
         "Action":[
            "ec2:ModifySnapshotAttribute",
            "ec2:CopySnapshot",
            "ec2:RegisterImage",
            "ec2:Describe*"
         ],
         "Resource":"*"
      }
   ]
}
' | sudo tee role-policy.json

```
 ## 7. Attach IAM Policy `role-policy.json` to IAM Role `vmimport`
 We'll attach the policy to IAM Role
 
 ```
 aws iam put-role-policy --role-name vmimport \
                        --policy-name vmimport \
                        --policy-document "file://role-policy.json"
 ```

## 8. Import VM Image
The following command will begin the import of the VM Image.

```
# Set the metadata, 
echo '[
  {
    "Description": "kalios",
    "Format": "vmdk",
    "UserBucket": {
        "S3Bucket": "'${bucket_name}'",
        "S3Key": "'${vm_image_name}'"
    }
}]
' > containers.json

```
Now after executing this command, you'll see `StatusMessage: "pending"`
The name of our VM Image is `containers.json`

Let's import this json file now

```
aws ec2 import-image --description "kalios" --disk-containers "file://containers.json"
```

*Output*
```
{
    "Description": "kalios",
    "ImportTaskId": "import-ami-0d1db1a95d061e4e3",
    "Progress": "2",
    "SnapshotDetails": [
        {
            "DiskImageSize": 0.0,
            "Format": "VMDK",
            "UserBucket": {
                "S3Bucket": "mig-VM-to-bucket",
                "S3Key": "VM-Import/vKaliOS-disk001.vmdk"
            }
        }
    ],
    "Status": "active",
    "StatusMessage": "pending"
}
```
We can see status still pending  `"StatusMessage": "pending"` 
Note down `ImportTaskId` to check the status of import job


## 9. Check the status of VM Image
Refer the `ImportTaskId` which you noted in previous step and write below mentioned command

```
aws ec2 describe-import-image-tasks --import-task-ids "import-ami-0d1db1a95d061e4e3"
```
*Output*

```
{
    "Description": "kalios",
    "ImportTaskId": "import-ami-0d1db1a95d061e4e3",
    "Progress": "28",
    "SnapshotDetails": [
        {
            "DiskImageSize": 877762318.0,
            "Format": "VMDK",
	    "Status": "active",
            "UserBucket": {
                "S3Bucket": "mig-VM-to-bucket",
                "S3Key": "VM-Import/vKaliOS-disk001.vmdk"
            }
        }
    ],
    "Status": "active",
    "StatusMessage": "converting"
}
```
This time status is `converting`
Wait until its completed....

## 10. Import Completed

```
[root:tmp]# aws ec2 describe-import-image-tasks --import-task-ids "import-ami-0d1db1a95d061e4e3"
```

```
{
    "ImportImageTasks": [
        {
            "Architecture": "x86_64",
            "Description": "kalios",
            "ImageId": "ami-0db67e8436107b5bc",
            "ImportTaskId": "import-ami-0d1db1a95d061e4e3",
            "LicenseType": "BYOL",
            "Platform": "Linux",
            "SnapshotDetails": [
                {
                    "Description": "kalios",
                    "DeviceName": "/dev/sda1",
                    "DiskImageSize": 877762318.0,
                    "Format": "VMDK",
                    "SnapshotId": "snap-0bc9d11c87634b2c3",
                    "Status": "completed",
                    "UserBucket": {
                        "S3Bucket": "mig-VM-to-bucket",
                        "S3Key": "VM-Import/vKaliOS-disk001.vmdk"
                    }
                }
            ],
            "Status": "completed"
        }
    ]
}
```
Here we go.... out status is Completed now

Now when you go to AMI, you will see new AMI created


## 11. Launch EC2 Machine from AMI
Now you can simply right click on the AMI and create EC2 machine from it


**Boom!! we have successfully migrated our on-prem VM to AWS **



