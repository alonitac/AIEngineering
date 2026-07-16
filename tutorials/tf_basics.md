# Infrastructure as Code with Terraform

**HashiCorp Terraform** is an Infrastructure as Code (IaC) tool that lets you define cloud resources in human-readable configuration files that you can version, reuse, and share.

![terraform_providers][terraform_providers]

## Motivation

Want to create exactly the same EC2 instance in 3 environments (dev, test, prod), or in 15 different regions? You can't really do it manually using the web console...
Using Terraform, it's as simple as configure the below code in a `.tf` file: 

```terraform
resource "aws_instance" "app_server" {
  ami           = "ami-0123456789abcdef0"
  instance_type = "t2.micro"

  tags = {
    Name = "some-instance"
    Terraform = "true"
  }
}
```

And perform the `terraform apply` command to provision the infrastructure in AWS. 

### IaC Benefits

- **Automated Provisioning**: Automates the creation and configuration of infrastructure, less human errors.
- **Consistent Environments**: Ensures uniformity across development, testing, and production environments.
- **Repeatable Process**: Allows for the replication of infrastructure setups, or reproduced infrastructure in a case of Disaster Recovery (RD).
- **Versioned and Documented**: IaC scripts are version-controlled, enabling teams to track changes over time and roll back to previous states if needed. 


## Install Terraform

https://learn.hashicorp.com/tutorials/terraform/install-cli?in=terraform/aws-get-started

> [!TIP]
> Install the [HashiCorp Terraform VS Code extension](https://marketplace.visualstudio.com/items?itemName=HashiCorp.terraform) for syntax highlighting, IntelliSense, and inline validation of `.tf` files.
> 
> Install the [HashiCorp agent skills](https://github.com/hashicorp/agent-skills) to get AI-assisted Terraform help in GitHub Copilot and other AI coding agents (code generation, module generation, provider development):
> ```bash
> npx skills add hashicorp/agent-skills
> ```

## Working with Terraform in AWS

Terraform creates and manages resources on cloud platforms through their APIs.

**Providers** are plugins that allow Terraform to interact with different platforms. 
Providers enable Terraform to create, read, update, and delete infrastructure resources in the platform you work with.
You can find all publicly available providers that Terraform can work with on the [Terraform Registry](https://registry.terraform.io/browse/providers), including AWS, Azure, GCP, Kubernetes, Helm, GitHub, Splunk, DataDog, and many more.


### Deploy a single EC2 instance

The set of files used to describe infrastructure in Terraform is known as a **Terraform configuration files** (`.tf`).

Let's provision a development EC2 instance for the PolyAI service. 

```terraform
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">=5.55"
    }
  }

  required_version = ">= 1.7.0"
}

provider "aws" {
  region  = "<aws-region>"
  profile = "default"  # change in case you want to work with another AWS account profile
}

resource "aws_instance" "polyai_dev" {
  ami           = "<ec2-ami>"
  instance_type = "t2.micro"

  tags = {
    Name = "<your-name>-polyai-dev"
    Env  = "dev"
  }
}

```

- The `terraform` block contains Terraform settings, including the required providers used to provision infrastructure.
  In this example, the aws provider's source is defined as `hashicorp/aws`.
- The `provider` block configures the specified provider, in this case `aws`.
- The `resource` blocks to define a physical or a virtual component in your infrastructure.
  In this example, the block declares a resource of type `aws_instance` (EC2 instance) with a local name `polyai_dev`.
  The resource local name is used internally by Terraform, but has no meaning in AWS. 


1. In your repo, create the file `infra/tf/main.tf`, copy the above code and change it as follows:
   1. `<aws-region-code>` is the region in which you want to deploy your infrastructure.
   2. `<ec2-ami>` is the AMI you want to provision (choose an Ubuntu AMI according to your region).
   3. `<your-name>` is your name, used to uniquely identify the instance.

   > [!NOTE]
   > All Terraform configuration files for this project should be placed under `infra/tf/` in your repository.

2. The directory containing your `.tf` files (`infra/tf/`) is known as the **Terraform workspace dir**.
   When first working with your TF workspace, you have to initialize it by running `terraform init` from within that directory. 

   Initializing a configuration directory downloads and installs the providers defined in the configuration, which in this case is the `aws` provider.
3. Run the `terraform plan` to create an execution plan, which lets you preview the changes that Terraform plans to make to your infrastructure.
4. Apply the configuration now with the `terraform apply` command.

When you applied your configuration, Terraform wrote data into a file called `terraform.tfstate`.
Terraform stores the IDs and properties of the resources it manages in this file, so that it can update or destroy those resources going forward.
The Terraform state file is **the only way** Terraform can track which resources it manages, and often **contains sensitive information**, so you must store your state file securely, outside your version control.


> [!NOTE]
> - You can make sure your configuration is syntactically valid and internally consistent by using the `terraform validate` command.
> - Inspect the current state using `terraform show`.
> - To list all resources in the state file, perform `terraform state list`.


### Destroy infrastructure

The `terraform destroy` command terminates resources managed by your Terraform project.

You can destroy specific resource by `terraform destroy -target RESOURCE_TYPE.NAME`.

# Exercises 

### :pencil2: Change the deployed infrastructure

1. Add another tag to the `aws_instance.polyai_dev` resource. Plan and apply. Can you see the updated EC2 in the AWS web console? What does **in-place** update mean? 
2. Change the instance type (e.g. from `t2.micro` to `t2.nano`). Plan and apply. Is terraform destroying your instance in order to perform the update?  
3. Update the `ami` of your instance. How many resources are going to be added or destroyed? Why?

### :pencil2: Extend the EC2

In `infra/tf/main.tf`, configure a Security Group resource:

```terraform
resource "aws_security_group" "polyai_dev_sg" {
  name        = "<your-name>-polyai-dev-sg"   # change <your-name> accordingly
  description = "Allow SSH and HTTP traffic"

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

Attach the SG by adding the following argument to the `aws_instance.polyai_dev` instance:

```diff
resource "aws_instance" "polyai_dev" {
  ...
+ security_groups = [aws_security_group.polyai_dev_sg.name]   
  ...
}
```

Similarly:

- Use the `aws_key_pair` resource to provision a Key-pair and use it in your instance.
- Use the `aws_ebs_volume` resource to create an 5GB EBS volume and attach it to your instance.

Terraform infers dependencies between resources based on the given configurations, so that resources are created (and destroyed) in the correct order. 
For example, the SG will be created first, then the EC2 instance itself.

Sometimes, there are hidden resource dependencies.
For example, your EC2 instance should read data from some S3 bucket. 
Without the bucket being provisioned first, the EC2 instance would not be functioning well. 

- Use the `aws_s3_bucket` resource to create an S3 bucket. Then use the `depends_on` argument to handle hidden resource dependencies that Terraform cannot automatically infer. 


### :pencil2: Manage the `terraform.tfstate` file 

As said, the state file keeps track of resources created by your configuration and maps them to real-world resources.
When you run `terraform apply`, Terraform writes metadata about your configuration to the state file and updates your infrastructure resources accordingly. 

Occasionally, you may need to manipulate your projects state outside of the standard workflow. 
This exercise addresses some common scenarios. 

#### Scenario I: Importing an existing resource into Terraform project

The `import` block can import an existing resources in AWS, which is not managed by Terraform, into your Terraform project.
Once imported, Terraform tracks the resource in your state file.

Let's import one of your existing EC2 instances into Terraform. 

1. In `main.tf`, add the following blocks:
   ```terraform
   import {  
    to = aws_instance.my_ec2
    id = "<instance-id>"
   }
   
   resource "aws_instance" "my_ec2" {  
     name = "my_ec2"
     # (other resource arguments...)
   }
   ```

2. Carefully plan and apply to import the resource into the state file.

The import block is **idempotent**. After importing, you can remove import blocks from your configuration files.

4. You should now configure the `aws_instance.my_ec2` resource to reflect the true configurations in AWS (the key pair name, VPC ID, attached security group name, etc...).
   To do so, you can either write it manually (the good and safe practice), or utilize the [configuration generator](https://developer.hashicorp.com/terraform/language/import/generating-configuration) tool of Terraform (Use carefully, it's an experimental tool). 
5. If you fully specify the configuration file according to the true state of the resource in AWS, you should see no plan when performing `terraform plan`, because the resource configuration reflects the actual state of the resource in AWS. 

#### Scenario II: Remove a resource from Terraform 

Use a `removed` block to remove specific resources from your state. 
This does not destroy the resource itself in AWS, instead it indicates that your Terraform configuration will no longer manage the resource.

1. Comment out the entire resource `aws_instance.my_ec2` block from `main.tf`.
2. Add a removed `block` to instruct Terraform to remove the resource from state, but not destroy it:
   ```terraform
   removed {  
    from = aws_instance.my_ec2 
    lifecycle {    
      destroy = false
    }
   }
   ```
3. Carefully plan and apply.
4. Confirm that Terraform forgot the resource by `terraform state list`, and that the instance still exists in AWS.

#### Scenario III: Resource drift

You should not make manual changes to resources controlled by Terraform, because the state file will be out of sync, or "drift," from the real infrastructure.
By default, Terraform compares your state file to real infrastructure whenever you invoke `terraform plan` or `terraform apply`.

If you suspect that your infrastructure configuration has changed outside of the Terraform workflow, you can use a `-refresh-only` flag to inspect what the changes to your state file would be.

1. Run `terraform plan -refresh-only` to determine the drift between your current state file and actual configuration.    
   You should be synced: `No changes. Your infrastructure still matches the configuration.`
2. In the AWS Console, **manually** add a new tag to one of your EC2 instances manages by Terraform.
3. Run `terraform plan -refresh-only` again. As shown in the output, Terraform has detected differences between the infrastructure and the current state.
4. Apply these changes by `terraform apply -refresh-only` to make your state file match your real infrastructure (**your `.tf` configuration file would not be updated accordingly, they remain un-synced**).

A refresh-only operation does not attempt to modify your infrastructure to match your Terraform configuration -- it only gives you the option to review and track the drift in your state file.
If you ran `terraform plan` or `terraform apply` without the `-refresh-only` flag now, Terraform would attempt to revert your manual changes.

5. Now, you should manually update your configuration accordingly.


[terraform_providers]: https://exit-zero-academy.github.io/DevOpsTheHardWayAssets/img/terraform_providers.png