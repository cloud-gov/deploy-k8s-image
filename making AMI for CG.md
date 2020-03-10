Process for making the AMI used by KOPS in CG

Binaries:

* Van's script
* AWS CLI
* Packer

Secrets:

* AWS Dev creds
* AWS govloud creds


* Run Van's import AMI script

This copies a public AMI for KOPs to our private AMI space in AWS Commercial.  The script though a series of AWS commands and S3 buckets (1 each in com and gov) makes an image export and image import into the private AMI on the govcloud side

* Run packer to update the new AMI with compliance and hardening tools

```
packer build <json build file>
```

The end output is the new AMI ID in govcloud private space for use in KOPS
