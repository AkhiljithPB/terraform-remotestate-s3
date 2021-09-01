# Storing terraform state (tfstate) in s3

Terraform uses persistent state data in a file named terraform.tfstate to keep track of the resources it manages. When working with Terraform in a team, use of a local file makes Terraform usage complicated because each user must make sure they always have the latest state data before running Terraform and make sure that nobody else runs Terraform at the same time.

With _remote state_, Terraform writes the state data to a remote data store, which can then be shared between all members of a team. Here I'm configuring an s3 bucket to store the terraform state for my project.

## Prerequisites.
-  Terraform.
-  IAM user with s3 service access.
-  S3 Bucket.


## Configuration.

I have already configured terraform for my project in a custom directory. 

```sh
[root@ip-172-31-8-216 project]# tree .terraform
.terraform
└── providers
    └── registry.terraform.io
        └── hashicorp
            └── aws
                └── 3.56.0
                    └── linux_amd64
                        └── terraform-provider-aws_v3.56.0_x5

6 directories, 1 file
[root@ip-172-31-8-216 project]#
```
I have created an s3 bucket "techhouse-tk" for my project and I'm setting the backend to s3

```sh
# cat backend.tf
terraform {
        backend "s3"{
        bucket      = "techhouse-tk"
        key         = "terraform/terraform.tfstate"
        region      = "ap-south-1"
        encrypt     = false
        }
}
```
Run "terraform init" with either the "-reconfigure" or "-migrate-state" flags to use the current configuration.
```sh
[root@ip-172-31-8-216 project]# terraform init -reconfigure

Initializing the backend...

Successfully configured the backend "s3"! Terraform will automatically
use this backend unless the backend configuration changes.

Initializing provider plugins...
- Reusing previous version of hashicorp/aws from the dependency lock file
- Using previously-installed hashicorp/aws v3.56.0

Terraform has been successfully initialized!

You may now begin working with Terraform. Try running "terraform plan" to see
any changes that are required for your infrastructure. All Terraform commands
should now work.

If you ever set or change modules or backend configuration for Terraform,
rerun this command to reinitialize your working directory. If you forget, other
commands will detect it and remind you to do so if necessary.
[root@ip-172-31-8-216 project]#
```
**NB : Make sure that you have configured aws on your machine with the IAM user having access to s3 service.**
- Once you have successfully configured the backend to s3 then generate a tfstate file to test the working.

```sh
[root@ip-172-31-8-216 project]# terraform apply

No changes. Your infrastructure matches the configuration.

Terraform has compared your real infrastructure against your configuration and found no differences, so no changes are needed.

Apply complete! Resources: 0 added, 0 changed, 0 destroyed.
[root@ip-172-31-8-216 project]#
```
You can see the file will be created under your s3bucket.
```sh
[root@ip-172-31-8-216 project]# aws s3 ls s3://techhouse-tk --recursive
2021-09-01 04:59:45          0 terraform/
2021-09-01 05:23:58        155 terraform/terraform.tfstate
[root@ip-172-31-8-216 project]#
```
Thank you.


## Referrence.
https://www.terraform.io/docs/language/settings/backends/configuration.html
