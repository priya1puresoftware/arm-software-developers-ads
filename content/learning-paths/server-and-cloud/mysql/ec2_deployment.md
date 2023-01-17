---
# User change
title: "Deploy single instance of MySQL"

weight: 2 # 1 is first, 2 is second, etc.

# Do not modify these elements
layout: "learningpathall"
---

##  Deploy single instance of MySQL 

## Prerequisites

* An [AWS account](https://portal.aws.amazon.com/billing/signup?nc2=h_ct&src=default&redirect_url=https%3A%2F%2Faws.amazon.com%2Fregistration-confirmation#/start)
* [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
* [AWS IAM authenticator](https://docs.aws.amazon.com/eks/latest/userguide/install-aws-iam-authenticator.html)
* [Ansible](https://www.cyberciti.biz/faq/how-to-install-and-configure-latest-version-of-ansible-on-ubuntu-linux/)
* [Terraform](https://github.com/zachlas/arm-software-developers-ads/blob/main/content/install-tools/terraform.md)
## Deploy EC2 instance via Terraform

To generate public key and private key through ```ssh-keygen```, use this [document](https://github.com/zachlas/arm-software-developers-ads/blob/main/content/learning-paths/server-and-cloud/aws/terraform.md).

![Screenshot (239)](https://user-images.githubusercontent.com/92315883/209257521-2b5e7019-1f4c-4701-a673-9d005b929893.png)


After generating the public and private keys, we have to create an EC2 instance. Then we will push our public key to the **authorized_keys** folder in `~/.ssh`. We will also create a security group that opens inbound ports `22`(ssh) and `3306`(MySQL). Below is a Terraform file called `main.tf` which will do this for us.

    

```console
provider "aws" {
  region = "us-east-2"
  access_key  = "AXXXXXXXXXXXXXXXXXXX"
  secret_key   = "AAXXXXXXXXXXXXXXXXXXXXXXXXXXX"
}
resource "aws_instance" "MYSQL_TEST" {
  ami           = "ami-064593a301006939b"
  instance_type = "t4g.small"
  security_groups= [aws_security_group.Terraformsecurity.name]
  key_name = "mysql_key"
  tags = {
    Name = "MYSQL_TEST"
  }

}

resource "aws_default_vpc" "main" {
  tags = {
    Name = "main"
  }
}
resource "aws_security_group" "Terraformsecurity" {
  name        = "Terraformsecurity"
  description = "Allow TLS inbound traffic"
  vpc_id      = aws_default_vpc.main.id

  ingress {
    description      = "TLS from VPC"
    from_port        = 3306
    to_port          = 3306
    protocol         = "tcp"
    cidr_blocks      = ["0.0.0.0/0"]
}
  ingress {
    description      = "TLS from VPC"
    from_port        = 22
    to_port          = 22
    protocol         = "tcp"
    cidr_blocks      = ["0.0.0.0/0"]
  }
  egress {
    from_port        = 0
    to_port          = 0
    protocol         = "-1"
    cidr_blocks      = ["0.0.0.0/0"]
  }

  tags = {
    Name = "Terraformsecurity"
  }

 }
resource "local_file" "inventory" {
    depends_on=[aws_instance.MYSQL_TEST]
    filename = "/home/ubuntu/inventory.txt"
    content = <<EOF
[all]
ansible-target1 ansible_connection=ssh ansible_host=${aws_instance.MYSQL_TEST.public_ip} ansible_user=ubuntu
                EOF
}

resource "aws_key_pair" "deployer" {
        key_name   = "mysql_key"
        public_key = "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCUZXm6T6JTQBuxw7aFaH6gmxDnjSOnHbrI59nf+YCHPqIHMlGaxWw0/xlaJiJynjOt67Zjeu1wNPifh2tzdN3UUD7eUFSGcLQaCFBDorDzfZpz4wLDguRuOngnXw+2Z3Iihy2rCH+5CIP2nCBZ+LuZuZ0oUd9rbGy6pb2gLmF89GYzs2RGG+bFaRR/3n3zR5ehgCYzJjFGzI8HrvyBlFFDgLqvI2KwcHwU2iHjjhAt54XzJ1oqevRGBiET/8RVsLNu+6UCHW6HE9r+T5yQZH50nYkSl/QKlxBj0tGHXAahhOBpk0ukwUlfbGcK6SVXmqtZaOuMNlNvssbocdg1KwOH ubuntu@ip-172-31-27-185"
 }
```
**NOTE:-** Replace `public_key`, `access_key`, `secret_key`, and `key_name` with original values.

Now, use the below Terraform commands to deploy the `main.tf` file.


### Terraform Commands

#### Initialize Terraform

Run `terraform init` to initialize the Terraform deployment. This command is responsible for downloading all dependencies which are required for the AWS provider.

```console
terraform init
```
    
![Screenshot (255)](https://user-images.githubusercontent.com/92315883/209255228-8c8b1b17-ce55-4c7d-9916-6c15918fc82e.png)


#### Create a Terraform execution plan

Run `terraform plan` to create an execution plan.

```console
terraform plan
```

**NOTE:** The **terraform plan** command is optional. You can directly run **terraform apply** command. But it is always better to check the resources about to be created.

#### Apply a Terraform execution plan

Run `terraform apply` to apply the execution plan to your cloud infrastructure. The below command creates all required infrastructure.

```console
terraform apply
```      
![Screenshot (264)](https://user-images.githubusercontent.com/92315883/209255731-6397eef8-d676-4053-941f-7bfe87e4972f.png)



## Configure MySQL through Ansible
Ansible is a software tool that provides simple but powerful automation for cross-platform computer support.
Ansible allows you to configure not just one computer, but potentially a whole network of computers at once.
Now to run Ansible, we have to create a `.yml` file, which is also known as `Ansible-Playbook`.
Playbook contains a collection of tasks.

### Here is the complete YML file of Ansible-Playbook
```console
---
- hosts: all
  remote_user: root
  become: true

  tasks:
    - name: Update the Machine
      shell: apt-get update -y
    - name: Installing Mysql-Server
      shell: apt-get -y install mysql-server
    - name: Installing PIP for enabling MySQL Modules
      shell: apt -y install python3-pip
    - name: Installing Mysql dependencies
      shell: pip3 install PyMySQL
    - name: start and enable mysql service
      service:
        name: mysql
        state: started
        enabled: yes
    - name: Change Root Password
      shell: sudo mysql -u root -e "ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY '{{Your_mysql_password}}'"
    - name: Create database user with password and all database privileges and 'WITH GRANT OPTION'
      mysql_user:
         login_user: root
         login_password: {{Your_mysql_password}}
         login_host: localhost
         name: Local_user
         host: '%'
         password: {{Give_any_password}}
         priv: '*.*:ALL,GRANT'
         state: present
    - name: Create a new database with name 'arm_test'
      community.mysql.mysql_db:
        name: arm_test
        login_user: root
        login_password: {{Your_mysql_password}}
        login_host: localhost
        state: present
        login_unix_socket: /run/mysqld/mysqld.sock
    - name: Copy database dump file
      copy:
       src: /home/ubuntu/table.sql
       dest: /tmp

    - name: Create a table with dummy values in database
      community.mysql.mysql_db:
       name: arm_test
       login_user: root
       login_password: {{Your_mysql_password}}
       login_host: localhost
       state: import
       target: /tmp/table.sql
    - name: MySQL secure installation
      become: yes
      expect:
        command: mysql_secure_installation
        responses:
           'Enter current password for root': '{{Your_mysql_password}}'
           'Set root password': 'n'
           'Remove anonymous users': 'y'
           'Disallow root login remotely': 'n'
           'Remove test database': 'y'
           'Reload privilege tables now': 'y'
        timeout: 1
      register: secure_mariadb
      failed_when: "'... Failed!' in secure_mariadb.stdout_lines"
    - name: Enable remote login by changing bind-address
      lineinfile:
         path: /etc/mysql/mysql.conf.d/mysqld.cnf
         regexp: '^bind-address'
         line: 'bind-address = 0.0.0.0'
         backup: yes
      notify:
         - Restart mysql
  handlers:
    - name: Restart mysql
      service:
        name: mysql
        state: restarted
```
**NOTE:-** We are using [table.sql]((https://github.com/Avinashpuresoftware/arm-software-developers-ads/files/10433773/table_dot_sql.txt)) script file to dump data, specify the `path` of the file accordingly. Replace `{{Your_mysql_password}}` and `{{Give_any_password}}` with your own password.

In our case, the inventory file will generate automatically. This file is formed after the `terraform apply` command. 

### Ansible Commands
To run a Playbook, we need to use the `ansible-playbook` command.
```console
ansible-playbook {your_yml_file} -i {your_inventory_file} --key-file {path_to_private_key}
```
**NOTE:-** Replace `{your_yml_file}`, `{your_inventory_file}` and `{path_to_private_key}` with orignal values.

![Screenshot (294)](https://user-images.githubusercontent.com/92315883/212034125-6f83ea1a-7a55-41b5-af66-a4677a4cdb0a.png)

Here is the output after the successful execution of the `ansible-playbook` command.

![Screenshot (237)](https://user-images.githubusercontent.com/92315883/209025480-02bd71ae-7a55-44ff-bc12-b4fa571c0663.png)


## Connect to Database using EC2 instance

To connect to the database, we need the `public-ip` of the instance where MySQL is deployed. We also need to use MySQL Client to interact with the MySQL database.

```console
apt install mysql-client
```

```console
mysql -h {public_ip of instance where Mysql deployed} -P3306 -u {user of database} -p {password of database}
```

**NOTE:-** Replace `{public_ip of instance where Mysql deployed}`, `{user_name of database}` and `{password of database}` with suitable values. In our case `user_name`= `Local_user`, which we have created through the `.yml` file. 

![Screenshot (297)](https://user-images.githubusercontent.com/92315883/212297155-e6983e10-ffb2-4a31-b2ec-ec626a2bda3b.png)


### Access Database and Table

We can access our database by using the below command.

```console
show databases;
```
![Screenshot (298)](https://user-images.githubusercontent.com/92315883/212295902-82b05825-a073-4d45-8273-6696dad7e9d8.png)

Tables can be accessed by using the below command.

```console
use {{your_database}};
```
```console
show tables;
```

![Screenshot (296)](https://user-images.githubusercontent.com/92315883/212296160-127d4b9e-47f2-4710-9498-9610e5a69c45.png)


