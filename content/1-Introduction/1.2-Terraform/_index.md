---
title : "What is Terraform?"
date :  "`r Sys.Date()`" 
weight : 2
chapter : false
pre : " <b> 1.1 </b> "
---
Currently, Terraform is probably the most common IaC (Infrastructure as Code) tool at the moment. It is an open-source tool by HashiCorp specifically used to create infrastructure through the code we write, instead of manually creating it in the console, which can be time-consuming.

![Terraform](/images/1.Introduction/terraform.png)

Other tools like Ansible can also achieve similar results, but Ansible is primarily a Configuration Management tool and not focused on IaC like Terraform. Therefore, using Ansible might require running unnecessary processes.

**Advantages:** 
- Open-source and free.
- Large community.
- Declarative programming.
- A major advantage: It can provide infrastructure for multiple cloud providers (AWS, GCP, Azure) in a single configuration file.

## Example of using Terraform:
Before implementing Terraform, we need to install the AWS CLI and Terraform CLI:
- Install AWS CLI. [AWS CLI Configuration](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-configure.html)
- Install Terraform CLI. [Terraform CLI Configuration](https://developer.hashicorp.com/terraform/tutorials/aws-get-started/install-cli)

![Terraform life cycle](/images/1.Introduction/01-tf.png)

To deploy infrastructure using Terraform, we need to know the steps and some common commands when using Terraform:
- First, we need to create a workspace, which can be a directory on the local machine.
- Next, write code to declare the infrastructure we want to deploy. As shown in the image below, we first need to declare the provider (in this case, "aws"), and create an EC2 instance of type "t2.micro" named "My EC2 Instance" in the Singapore region.

![Terraform example](/images/1.Introduction/02-tf.png)

To initialize the workspace, run the command **terraform init** to download the AWS provider to the current workspace so Terraform can use these providers to call APIs and create resources for us. After running the command, the `.terraform` directory will appear in the workspace, containing the code of the declared providers.

![Terraform workspace](/images/1.Introduction/03-tf.png)

{{% notice info %}}
Here, we can run the command **terraform fmt** to format the Terraform files.
{{% /notice %}}

{{% notice info %}}
Additionally, to check the validity of the code we wrote, we can use the command **terraform validate**.
{{% /notice %}}

After initializing the workspace with **terraform init**, run the command **terraform plan** to see which resources will be created/modified/deleted. ⇒ Create 1 resource, 0 modify, 0 delete.
![Terraform output](/images/1.Introduction/04-tf.png)

After reviewing the plan, run the command **terraform apply** to execute those actions.

{{% notice warning %}}
Avoid manually modifying or deleting resources created by Terraform, as it will disrupt the state management of the resources controlled by Terraform.
{{% /notice %}}

{{% notice tip %}}
If, after applying and creating the resource, there are errors and you want to fix them, you can edit the code and run **terraform apply** again. Terraform will detect the changes and update the resource without needing to destroy it and start over.
{{% /notice %}}

{{% notice tip %}}
After running **terraform plan**, if you modify the code, **apply** will run the new code. If you want to apply the code as it was when **terraform plan** was executed, use “-out=<filename>”. Then run **terraform apply <filename>**.
{{% /notice %}}

![Terraform result](/images/1.Introduction/05-tf.png)

After running **terraform apply**, the EC2 instance named "My EC2 Instance" of type "t2.micro" has been created.
![Terraform result](/images/1.Introduction/06-tf.png)

Finally, if you want to delete the resource, simply run the command “terraform destroy”. Terraform will delete all the resources that were deployed from the start.

⇒ In summary, Terraform can be understood as a tool that manages resource state through the `terraform.tfstate` file created when running the “terraform apply” command. It also helps perform CRUD actions on the infrastructure resources we want.
