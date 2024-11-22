# ami-export


Ensure S3 Bucket Configuration
Create the S3 bucket (if not already created) in your desired region (e.g., Mumbai):
bash
Copy code
aws s3api create-bucket --bucket testing-for-snapshots5 --region ap-south-1 --create-bucket-configuration LocationConstraint=ap-south-1
Enable ACLs and set Object Ownership to Bucket owner preferred.
Edit the bucket's ACL to add permissions:
Grantee: AWS service (VM Import/Export).
Permissions: Object - Write and Bucket ACL - Read.
Example bucket policy:

json
Copy code
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Service": "vmie.amazonaws.com"
            },
            "Action": [
                "s3:PutObject",
                "s3:GetObject",
                "s3:GetBucketLocation",
                "s3:ListBucket"
            ],
            "Resource": [
                "arn:aws:s3:::testing-for-snapshots5",
                "arn:aws:s3:::testing-for-snapshots5/*"
            ]
        }
    ]
}
Attach this policy to the bucket:

bash
Copy code
aws s3api put-bucket-policy --bucket testing-for-snapshots5 --policy file://bucket-policy.json
2. IAM Configuration
Create the vmimport Role:
Create the trust policy:
bash
Copy code
cat > trust-policy.json <<EOF
{
   "Version": "2012-10-17",
   "Statement": [
      {
         "Effect": "Allow",
         "Principal": { "Service": "vmie.amazonaws.com" },
         "Action": "sts:AssumeRole"
      }
   ]
}
EOF
Create the role:
bash
Copy code
aws iam create-role --role-name vmimport --assume-role-policy-document file://trust-policy.json
Attach Permissions to the vmimport Role:
Create a role policy:
bash
Copy code
cat > role-policy.json <<EOF
{
   "Version":"2012-10-17",
   "Statement":[
      {
         "Effect": "Allow",
         "Action": [
            "s3:GetBucketLocation",
            "s3:GetObject",
            "s3:ListBucket",
            "s3:PutObject",
            "s3:GetBucketAcl"
         ],
         "Resource": [
            "arn:aws:s3:::testing-for-snapshots5",
            "arn:aws:s3:::testing-for-snapshots5/*"
         ]
      },
      {
         "Effect": "Allow",
         "Action": [
            "ec2:ModifySnapshotAttribute",
            "ec2:CopySnapshot",
            "ec2:RegisterImage",
            "ec2:Describe*",
            "ec2:ExportImage"
         ],
         "Resource": "*"
      }
   ]
}
EOF
Attach the policy to the role:
bash
Copy code
aws iam put-role-policy --role-name vmimport --policy-name vmimport-policy --policy-document file://role-policy.json
3. Export the AMI
Run the export command:
bash
Copy code
aws ec2 export-image \
    --image-id ami-056dd41bb3180dba8 \
    --disk-image-format VMDK \
    --s3-export-location S3Bucket=testing-for-snapshots5,S3Prefix=exports/
4. Monitor the Export Task
Use the export task ID from the output and describe the task to check its status:
bash
Copy code
aws ec2 describe-export-image-tasks --export-image-task-ids <export-task-id>
Wait for the task to complete. The statusMessage will show progress.
5. Verify the Export
Once the task is complete, the exported VMDK will be available in your S3 bucket under the specified prefix (exports/).

