# Storing terraform state (tfstate) in s3

Terraform uses persistent state data in a file named terraform.tfstate to keep track of the resources it manages. When working with Terraform in a team, use of a local file makes Terraform usage complicated because each user must make sure they always have the latest state data before running Terraform and make sure that nobody else runs Terraform at the same time.

With _remote state_, Terraform writes the state data to a remote data store, which can then be shared between all members of a team. Here I'm configuring an s3 bucket to store the terraform state for my project.

## Prerequisites.
-  Terraform.
-  S3 Bucket.

# steps:

### Creating our s3 bucket using terraform.

Here, I'm creating a new file in my working directory labeled s3.tf

```sh
resource "aws_s3-bucket" "remote_bucket" {
        bucket          =       "techhouse-tk"
        acl             =       "private"
    versioning {
        enabled         =       true
     }
    tags = {
        Name            =       "techhouse-tk"
    }
}
```
##### State Lock:

Terraform will lock your state for all operations that could write state. This prevents multiple people attempting to make change to same file which can cause damage or data loss.
You can disable state locking for most commands with the _-lock_ flag but it is not recommended.

### Creating the DynamoDB Table.
I have created a new file in my working directory labeled dynamo.tf for the resource "aws_dynamodb_table".

```sh
resource "aws_dynamodb_table" "dynamodb-tf-state-lock" {
        name            = "tf-state-lock-dynamo"
        hash_key        = "LockID"
        read_capacity   = 5
        write_capacity  = 5

  attribute {
        name            = "LockID"
        type            = "S"
  }
depends_on              = [aws_s3_bucket.remote_bucket]
}
```
# What is DynamoDB?
 - https://youtu.be/sI-zciHAh-4

After creating both the resources, S3 bucket and DynamoDB table, we have to modify our terraform s3 backend to add bucket_name and table_name.

### Setting up the Backend to use S3.
I have created a file in the working directory labeled backend.tf

```sh
terraform {
  backend "s3" {
    encrypt         = true
    bucket          = "techhouse-tk"
    dynamodb_table  = "tf-state-lock-dynamo"
    key             = "terraform.tfstate"
    region          = "ap-south-1"
  }
}
```
Note : _Before running the s3 backend script , we must create the two resources (i.e. s3 bucket and DynamoDB table)_

###Lets now initialize the script to put it all together.
```sh
[root@ip-172-31-8-216 remote-tf]# terraform init

Initializing the backend...
Successfully configured the backend "s3"! Terraform will automatically
use this backend unless the backend configuration changes.

Initializing provider plugins...
- Reusing previous version of hashicorp/aws from the dependency lock file
- Using previously-installed hashicorp/aws v3.56.0

Terraform has been successfully initialized!
```
```sh
[root@ip-172-31-8-216 remote-tf]# terraform apply
╷
│ Error: Error acquiring the state lock
│
```
### Solving State Lock Error.
State locking happens automatically on all operations that could write state.
To move forward with your apply use the below command,
```sh
[root@ip-172-31-8-216 remote-tf]# terraform apply -lock=false
  Enter a value: yes
aws_dynamodb_table.dynamo-tf-state-lock: Creating...
aws_dynamodb_table.dynamo-tf-state-lock: Creation complete after 7s [id=tf-state-lock-dynamo]
Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
[root@ip-172-31-8-216 remote-tf]#
```
Inspect the .terraform/terraform.tfstate to verify the state lock with DynamoDB.

```sh
[root@ip-172-31-8-216 remote-tf]# cat .terraform/terraform.tfstate |grep -E 'dynamo|bucket'
            "bucket": "techhouse-tk",
            "dynamodb_endpoint": null,
            "dynamodb_table": "tf-state-lock-dynamo",
[root@ip-172-31-8-216 remote-tf]#
```
Check your Dynamo DB table items from the AWS console. You can see the .tfstate file on Dynamodb with a Lock string.

![alt text](https://github.com/AkhiljithPB/terraform-remotestate-s3/blob/af26b53f741f47d22a5bb2a948b0d6b83597ca76/lockstring.png?raw=true)
