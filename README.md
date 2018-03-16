[![Build Status](https://travis-ci.org/OlegGorj/vpc-with-bastionbox.terraform.svg?branch=master)](https://travis-ci.org/OlegGorj/vpc-with-bastionbox.terraform)
[![CircleCI](https://circleci.com/gh/OlegGorj/vpc-with-bastionbox.terraform/tree/master.svg?style=svg)](https://circleci.com/gh/OlegGorj/vpc-with-bastionbox.terraform/tree/master)


# Repository vpcs-with-bastionbox.terraform

Modular Terraform repository to provision a multi-tier VPC in AWS. By default it will create:

- One public and one private subnets in one AZ for the chosen region
- Internal DNS zone associated to the VPC for internal domain resolution (DNS name is autogenerated)
- Internet gateway for the public subnets
- One EC2 NAT gateway per AZ for the private subnets
- One routing table per private subnet associated to the corresponding EC2 NAT gateway
- One BastionBox without internal DNS zone record (eg. DNS name is autogenerated)
- One OpenVpn box in public subnet to manage SSH access to all nodes

Since it is modular it is easy to add or remove modules depending on preferences and requirements.

## Methodology

The goal of this terraform project is to create code that can be used in across multiple environments (such as dev, staging, prod). To do this, it uses the environments folder to define the variables for each environment. The naming convention that I use is <root>/terraform/environments/$ENV/inputs.tfvars. Where `ENV` can be `dev`, `qa`, `stg`, `prod`, `POC-of-some-sort`, etc.

## Variables

The variables that are needed are set in the environments directory. This allows you to setup a different VPCs for different environments (i.e. dev, stage, or prod). The variables that need to be set are:

```
# AWS Informatation
env               = "dev"                           # environment
aws_region        = "us-west-1"                     # The AWS Account Region

# aws_vpc Module
vpc_name          = "vpc_dev"                      # Name for your VPC
vpc_cidr          = "10.233.0.0/16"                # CIDR for your VPC
r53_zone_name     = "dev.mydomainname.com"         # Hosted Zone for Route 53

# public-subnets module
public-1_create = true                             # Boolean to determine whether or not to build each public subnet
# public-subnets module
private-1_create = false                            # Boolean to determine whether or not to build each private subnet

public-1_subnet_cidr = "10.233.0.0/28"             # CIDR for public Subnet
private-1_subnet_cidr = "10.233.1.0/28"            # CIDR for Private Subnet

```


### Deployment:

![](deployment.jpg)

### Prerequisites

1. First things first - how to install Terraform on you Mac or Win, follow this [link](https://www.terraform.io/intro/getting-started/install.html)

2. Create AWS S3 bucket to manage Terraform state. This example uses bucket named `aws-terraform-state-bucket`



### Step-by-step instructions:

#### 1.  Go into 'terraform' folder.

```
cd ~/terraform
```

#### 2.  Create the keys pair for the dev servers (at this point, assume `DEV` environment setup)

```
ssh-keygen -t rsa -C "dev_key" -f ~/.ssh/dev_key

```

#### 3.  Specify things like Access and secret key in some ways:

*Option 1* - Specify it directly in the provider (not recommended)

```
provider "aws" {
  region     = "us-west-1"
  access_key = "myaccesskey"
  secret_key = "mysecretkey"
}
```
Obviously, it has some downsides, like, you would need to put your credentials into TF script, which is very bad idea, hence it's highly NOT recommended.


*Option 2* - Using the shared_credentials_file to pass environment credentials

```
provider "aws" {
  region = "${var.region}"
  shared_credentials_file  = "${var.cred-file}"
}

```

where variable `${var.cred-file}` looks like:

```
variable "cred-file" {
  default = "~/.aws/credentials"
}

```

Node: `~/.aws/credentials` points to credentials file located in your home directory. For development purposes, this might be fine, but for PROD deployment, this will needs to be replaced with call to Vault.

File `~/.aws/credentials` has following format:

```
[default]
aws_access_key_id = <your access key>
aws_secret_access_key = <your secret key>
```

Of course, there are bunch of other options to manage secrets and keys, but this is not the objective of this repo (although, it's on TODO list).

The second option is *recommended* because you don’t need to expose your secrets on TF script. And, again, proper integration with the vault and KMS is on my TODO.

Hence, `_main.tf` would look like:

```
provider "aws" {
  region = "${var.region}"
  shared_credentials_file  = "${var.cred-file}"
}

resource "aws_key_pair" "key" {
  key_name   = "${var.key_name}"
  public_key = "${file("dev_key.pub")}"
}
```

#### 4. Directory structure would look like this:

```
| => tree
.
├── Makefile
├── README.md
├── _config.yml
├── deploy.sh
├── deployment.jpg
├── terraform
│   ├── _main.tf
│   ├── backend.tf
│   ├── environments
│   │   ├── dev
│   │   │   └── inputs.tfvars
│   │   └── prod
│   ├── modules
│   │   ├── aws-vpc
│   │   │   └── main.tf
│   │   ├── bastion
│   │   │   └── main.tf
│   │   ├── networking
│   │   │   └── main.tf
│   │   ├── openvpn
│   │   │   └── main.tf
│   │   ├── privider
│   │   │   └── main.tf
│   │   ├── public-subnets
│   │   └── web
│   │       ├── files
│   │       │   └── user_data.sh
│   │       ├── main.tf
│   │       └── variables.tf
│   └── variables.tf

```

#### 5. Run this command on terraform folder: (Terraform’s commands should be run on the environments folder).

Assuming, bucket with the name `aws-terraform-state-bucket` already created.

```
=> cd ~/terraform

=> terraform init -backend-config="bucket=aws-terraform-state-bucket" -backend-config="key=vpc-with-bastionbox.tfstate" -backend-config="region=us-west-1" -var-file=./environments/dev/inputs.tfvars

=> terraform get
=> terraform plan

```

You should see a lot of output ending with this

```
Plan: 21 to add, 0 to change, 0 to destroy.

```

#### 6.  and, finally, apply the changes

```
=> terraform apply

```

After a while you should see something similar to this...

```

Apply complete! Resources: 21 added, 0 changed, 0 destroyed.

Outputs:

elb_hostname = dev-web-lb-XXXXXXX.us-west-1.elb.amazonaws.com
```

#### 7.  Access Web server via LB

If you run command `terraform output elb_hostname`, you should get public DNS address of LB

```
dev-web-lb-XXXXXXX.us-west-1.elb.amazonaws.com
```

Copy and paste it into the browser window, specify port 80

```
dev-web-lb-XXXXXXX.us-west-1.elb.amazonaws.com:80
```

You should see Nginx welcome screen.
If you'd like to play around with Nginx home page, make changes to shell script `terraform/modules/web/files/user_data.sh`


#### 8. OpenVPN

At the end of execution of `terraform apply`, you should see something similar to the following lines, the result of execution of entire module:

```
aws_vpn_instance_public_dns = ec2-13-xx-xxx-xx.us-west-1.compute.amazonaws.com
aws_vpn_instance_public_ip = 13.xx.xxx.xx
client_configuration_file = docker.ovpn
closing_message = Your VPN is ready - check out client configuration file to configure your client. '
elb_hostname = dev-web-lb-XXXXXXX.us-west-1.elb.amazonaws.com

```

VPN client config `docker.ovpn` file will be copied to your home directory.

Publich DNS name of VPN is indicated by variable `aws_vpn_instance_public_dns`


#### 9. VPN client

Use generated file `docker.ovpn` with an OpenVPN client. In OS X, you can install openvpn using command `brew install openvpn`.

Then, once that is done,

```
$ sudo openvpn --config awesome-personal-vpn.ovpn

```

Alternatively, use GUI client Tunnelblick at [this link](https://openvpn.net/index.php/access-server/docs/admin-guides/183-how-to-connect-to-access-server-from-a-mac.html)

Once Tunnelblick is installed, import `docker.ovpn` file.

#### 10. SSH to Web nodes

Once you login to VPN, you should be able to see all nodes in VPC and you can connect to Web servers (you can find Web private IPs using Console).

Web node 1:

```
ssh ubuntu@10.0.2.182
```

Web node 2:

```
ssh ubuntu@10.0.2.33
```


#### 11. Destroy everything

And the last step is to destroy all setup


```
=> terraform destroy
```

---


## TODOs

- use docker to properly deploy Ngnix
- integration with Vault and KMS to manage secrets and keys
- create clean Makefile to support deployment with Travis
- add autoscale group for Web nodes
- reorganize code structure into modular for simpler deployment


---
