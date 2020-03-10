# Process for making the AMI used by KOPS in CG
## Requirements

Binaries:

* AWS CLI
* Packer

Secrets:

* AWS Commerical creds
* AWS GovCloud creds


## Steps
### Migrate AMI from AWS Commerical to AWS GovCloud
Before we can do VM hardening we need to grab KOPS AMI from AWS Com and bring it to AWS GovCloud

Notes:
* KOPS images get regularly updated with security fixes in Commerical but not GovCloud
* Packer itself supports copying AMIs to different region but due to AWS restrictions of Com/GovCloud being separate entities.

##### 1. Copy AMI to account
You can't export image if it's not under your account so need to take a copy of the image first
```
aws ec2 copy-image --source-image-id ami-03bd8 --source-region us-west-2 --name kops-copy-to-account --profile aws-com
```

Note: Can use AMI Concourse resource to monitor new releases of images

##### 2. Export AMI as vmdk and save in S3
Now that AMI is on your account, you can export the image to your S3 Bucket 
```
aws ec2 export-image --image-id ami-06e2f --disk-image-format vmdk  --s3-export-location S3Bucket=<bucket-in-com>,S3Prefix=kops-test-march-9 --profile aws-com
```

Note: Can use AMI Concourse resource to monitor when AMIs has been copied

##### 3. Copy VMDK to S3
Next, upload the VMDK to the AWS GovCloud S3 Bucket so it be used for import

Note: Could use S3 Concourse Resource if using Concourse.

##### 4. Import VMDK as Snapshot to Gov Cloud
The reason why you need to import as snapshot is so you can get around AWS restrictions that Amazon put on `import-image` so Debian 9.0 or 10.0 does not work.
```
aws ec2 import-snapshot --disk-container file://import.json --profile aws-dev-gov
```
Example: `import.json`
```
{
      "Description": "KOPS AMI import OVA",
      "Format": "vmdk",
      "UserBucket": {
          "S3Bucket": "<GOV-CLOUD-S3>",
          "S3Key": "<NAMEOFFILEINGOVS3.vmdk>"
      }
}
```

For Reference of OS requirements: https://docs.aws.amazon.com/vm-import/latest/userguide/vmie_prereqs.html#prerequisites


### 5. Register Snapshot as AMI
Lastly, register the image from the snapshot!
```
aws ec2 register-image --name kops-image --block-device-mappings file://register.json
```

Example: `register.json`
```
[
  {
    "DeviceName": "string",
    "VirtualName": "string",
    "Ebs": {
      "DeleteOnTermination": true,
      "Iops": integer,
      "SnapshotId": "snapshot-id",
      "VolumeSize": 8,
      "VolumeType": "standard",
      "KmsKeyId": "string",
      "Encrypted": true
    },
    "NoDevice": "string"
  }
]
```


## Use Packer to add VM hardening
* Run packer to update the new AMI with compliance and hardening tools

```
packer build <json build file>
```

The end output is the new AMI ID in govcloud private space for use in KOPS

## Other Notes and Thoughts
* Packer Post Processors - `Amazon-import` does same thing as `aws ec2 import-image`
* Packer Builder `Amazon-ebs` can only accept one set of creds 
* Packer region validations need to be turned off for Gov Cloud most of the time
