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


After generating the public and private keys now, we have to create an EC2 instance. And we will push our public key to the **authorized_keys** folder of `.ssh`.
We will also create one security group and open the inbound ports. We need to open ports `22`(ssh) for ssh connection and `3306`(MySQL). For the creation of the instance first, we need to create a `main.tf` file.


### Here is the complete main.tf file
    

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
  provisioner "local-exec" {
    command = "echo ${self.private_ip} >> private_ips.txt && echo ${self.public_ip} >> public_ips.txt && echo ${self.public_dns} >> public_ips.txt"
  }

  connection {
    type        = "ssh"
    host        = self.public_ip
    user        = "ubuntu"
    private_key = file("~/.ssh/mysql_key")
    timeout     = "4m"
  }
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
resource "aws_key_pair" "deployer" {
        key_name   = "mysql_key"
        public_key = "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCUZXm6T6JTQBuxw7aFaH6gmxDnjSOnHbrI59nf+YCHPqIHMlGaxWw0/xlaJiJynjOt67Zjeu1wNPifh2tzdN3UUD7eUFSGcLQaCFBDorDzfZpz4wLDguRuOngnXw+2Z3Iihy2rCH+5CIP2nCBZ+LuZuZ0oUd9rbGy6pb2gLmF89GYzs2RGG+bFaRR/3n3zR5ehgCYzJjFGzI8HrvyBlFFDgLqvI2KwcHwU2iHjjhAt54XzJ1oqevRGBiET/8RVsLNu+6UCHW6HE9r+T5yQZH50nYkSl/QKlxBj0tGHXAahhOBpk0ukwUlfbGcK6SVXmqtZaOuMNlNvssbocdg1KwOH ubuntu@ip-172-31-27-185"
 }
```
**NOTE:-** Replace `public_key`, `access_key`, `secret_key` and `key_name` with orignal values. Also replace `private_key` with correct path.

Now, use below Terraform commands to deploy `main.tf` file.


### Terraform Commands

#### Initialize Terraform

Run `terraform init` to initialize the Terraform deployment. This command is responsible for downloading all dependencies which are required for the provider AWS.

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
**NOTE:-** We are using [table.sql](https://github.com/Avinashpuresoftware/arm-software-developers-ads/files/10311465/table_dot_sql.txt) script file to dump data, specify the `path` of file accordingly. Replace `{{Your_mysql_password}}` and `{{Give_any_password}}` with your own password.

### Create inventory file.

We also need the **inventory.txt** file. The Ansible inventory file defines the hosts and groups of hosts upon which commands, modules, and tasks in a playbook operate.
#### Here is the inventory.txt file

```console
[all]
ansible-target1 ansible_host={{Use private-ip of instance}}  ansible_connection=ssh ansible_user={{Enter user}}
```
**NOTE:-** Use private-ip of instance(where you want to config MySQL) as `ansible_host` and user as `ansible_user`. 

Private-IP of instance will be present in `private_ips.txt` file. This file is formed after `terraform apply` command. 


### Ansible Commands
To run a Playbook, we need to use the `ansible-playbook` command.
```console
ansible-playbook {your_yml_file} -i {your_inventory_file} --key-file {path_to_private_key}
```
**NOTE:-** Replace `{your_yml_file}`, `{your_inventory_file}` and `{path_to_private_key}` with orignal values.

![Screenshot (236)](https://user-images.githubusercontent.com/92315883/209024640-e9a9403c-481b-4232-848e-31d8e033c092.png)

Here is the output after the successful execution of the `ansible-playbook` command.

![Screenshot (237)](https://user-images.githubusercontent.com/92315883/209025480-02bd71ae-7a55-44ff-bc12-b4fa571c0663.png)

