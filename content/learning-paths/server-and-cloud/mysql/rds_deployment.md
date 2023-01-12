---
# User change
title: "Deploy MySQL using RDS"

weight: 3 # 1 is first, 2 is second, etc.

# Do not modify these elements
layout: "learningpathall"
---
## Prerequisites

* An [AWS account](https://portal.aws.amazon.com/billing/signup?nc2=h_ct&src=default&redirect_url=https%3A%2F%2Faws.amazon.com%2Fregistration-confirmation#/start)
* [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
* [AWS IAM authenticator](https://docs.aws.amazon.com/eks/latest/userguide/install-aws-iam-authenticator.html)
* [Terraform](https://github.com/zachlas/arm-software-developers-ads/blob/main/content/install-tools/terraform.md)

## Deploy MySQL RDS instances

RDS is a Relational database service provided by AWS. More information can be found [here]((https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/CHAP_GettingStarted.CreatingConnecting.MySQL.html).) To deploy a MySQL RDS instance, we need to create a `main.tf` Terraform file

### Here is the complete main.tf file
    

```console

provider "aws" {
  region = "us-east-2"
  access_key  = var.access_key # access through credential file. 
  secret_key   = var.secret_key
}

resource "aws_db_instance" "Testing_Mysql" {
  identifier           = "mysqldatabase"
  allocated_storage    = 10
  db_name              = "mydb"
  engine               = "mysql"
  engine_version       = "8.0.28"
  instance_class       = "db.t3.micro"
  parameter_group_name = "default.mysql8.0"
  skip_final_snapshot  =  true
  username              = var.username
  password              = var.password
  availability_zone     = "us-east-1a"
  publicly_accessible  = true
  deletion_protection   = false

  tags = {
        name                 = "TEST MYSQL"
  }

}

``` 

To find the correct instance type for RDS, Check the [list](https://aws.amazon.com/rds/mysql/instance-types/) of supported instance types. We selected a Graviton (Arm) based instance type.

![Screenshot (260)](https://user-images.githubusercontent.com/92315883/209249327-3755d7ef-581b-456c-a64b-e2167080dd59.png)
We also need to create `credentail.tf` file, for passing our secret keys and password.

### Here is credential.tf file for credentials

```console
variable "username"{
      default  = "admin"
}

variable "password"{
      default  = "Armtest"    #we_can_chosse_any_password, except special_characters.
}

variable "access_key"{
      default  = "AKIXXXXXXXXXXXXXX"
}

variable "secret_key"{
      default  = "EXXXXXXXXXXXXXXXXXXXXXXXXXXX"
}


```
**NOTE:** Replace `secret_key` and `access_key` with orignal AWS credentials.
Now, use below Terraform commands to deploy `main.tf` file.

## Terraform commands
    
### Initialize Terraform

Run `terraform init` to initialize the Terraform deployment. This command is responsible for downloading all dependencies which are required for the AWS provider.


```console
terraform init
```
![Screenshot (255)](https://user-images.githubusercontent.com/92315883/209247057-71265c2d-e52a-411c-91f2-e774d51874bb.png)

### Create a Terraform execution plan

Run `terraform plan` to create an execution plan.

```console
terraform plan
```

**NOTE:** The **terraform plan** command is optional. You can directly run **terraform apply** command. But it is always better to check the resources about to be created.

### Apply a Terraform execution plan

Run `terraform apply` to apply the execution plan to your cloud infrastructure. Below command creates all required infrastructure.

```console
terraform apply
```      

![Screenshot (256)](https://user-images.githubusercontent.com/92315883/209247083-91a719df-8707-4380-9637-d1238cacf8b3.png)
   
### Verify the RDS setup
   
Verify the setup by going back to the AWS console. Go to **RDS » Databases**, you should see the instance running.  

![Screenshot (257)](https://user-images.githubusercontent.com/92315883/209247626-2df854ca-a781-46b0-aeba-076a23b0c1fb.png)

## Connect to RDS using EC2 instance

To access the RDS instance, we need to make sure that our instance is correctly associated with a security group and VPC. To access RDS outside the VPC, Follow this [document](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/CHAP_CommonTasks.Connect.html).

To connect to the RDS instance, we need the `Endpoint` of the RDS instance. To find the Endpoint, Go to **RDS »Dashboard » {{YOUR_RDS_INSTANCE}}**.

![Screenshot (280)](https://user-images.githubusercontent.com/92315883/209741254-55b40b52-1c56-482a-ab48-e33f510a1cf6.png)

Now, we can connect to RDS using the above Endpoint. And we also have to use the `user` and `password` mentioned in `credential.tf` file.

```console
mysql -h {{Endpoint}} -u {{user}} -p {{password}}
```
**NOTE:** Replace `{{Endpoint}}`, `{{user}}` and `{{password}}`  with suitable values.

![Screenshot (277)](https://user-images.githubusercontent.com/92315883/209741354-7872aac9-97cd-4554-ade8-80f8a4bbdf25.png)


### Create Database and Table
Create `arm_test` database as shown below.

![Screenshot (278)](https://user-images.githubusercontent.com/92315883/209741488-8e4cc2f6-e3d0-4730-8321-db2a21cc27c2.png)

Create a table and insert values by using the script file [table.sql](https://github.com/Avinashpuresoftware/arm-software-developers-ads/files/10311465/table_dot_sql.txt).

**NOTE:** Make sure the `table.sql` file should be present in the same directory where `main.tf` and `credential.tf` are.

![Screenshot (279)](https://user-images.githubusercontent.com/92315883/209741539-9cc80fae-7fb1-4eae-b1c2-17235e48630d.png)

