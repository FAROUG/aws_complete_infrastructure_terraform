# terrafrom
### Prerequists
#### terminology
Existing Account --> an aws account that is running same project you are deploying such as development, staging, testing, UAT, or PrePord account.
Target Account --> the count in which you are planning to replicate the project such as production account.
AMI --> an ami with the prerequist configuration ( in this case with the ssh access to the bitbucket repositories)
* in case of initail deployment to the project prepare an AMI 
* in case of replication of the project move the AMI from the Existing Account to the Target Account
### Key Pair Section
#### Terraform can generate SSL/SSH private keys using the tls_private_key resource.

So if you wanted to generate SSH keys on the fly you could do something like this:
```
variable "key_name" {}

resource "tls_private_key" "example" {
  algorithm = "RSA"
  rsa_bits  = 4096
}

resource "aws_key_pair" "generated_key" {
  key_name   = var.key_name
  public_key = tls_private_key.example.public_key_openssh
}

data "aws_ami" "ubuntu" {
  most_recent = true

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-focal-20.04-amd64-server-*"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }

  owners = ["099720109477"] # Canonical
}

resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t2.micro"
  key_name      = aws_key_pair.generated_key.key_name

  tags {
    Name = "HelloWorld"
  }
}

output "private_key" {
  value     = tls_private_key.example.private_key_pem
  sensitive = true
}
```
This will create an SSH key pair that lives in the Terraform state (it is not written to disk in files other than what might be done for the Terraform state itself when not using remote state), creates an AWS key pair based on the public key and then creates an Ubuntu 14.04 instance where the ubuntu user is accessible with the private key that was generated.

You would then have to extract the private key from the state file and provide that to the users. You could use an output to spit this straight out to stdout when Terraform is applied.

Getting the output from private key is via this command below:

```terraform output -raw private_key > ~/.ssh/terraform.keypair.pem | chmod 600 ~/.ssh/terraform.keypair.pem```

#### Security caveats
I should point out here that passing private keys around is generally a bad idea and you'd be much better having developers create their own key pairs and provide you with the public key that you (or them) can use to generate an AWS key pair (potentially using the aws_key_pair resource as used in the above example) that can then be specified when creating instances.

In general I would only use something like the above way of generating SSH keys for very temporary dev environments that you are controlling so you don't need to pass private keys to anyone. If you do need to pass private keys to people you will need to make sure that you do this in a secure channel and that you make sure the Terraform state (which contains the private key in plain text) is also secured appropriately.

([Reference](https://stackoverflow.com/questions/49743220/how-do-i-create-an-ssh-key-in-terraform))
#### change concept
##### bitbucket access
the first implimination for bitbuct access was to creat an instance that will  generate a pair of ssh keys to be used for the access at the runtime and display them as terraform output, but that require to refer the created ami id to the autoscaling resources.
however easy implimintation is to creat an s3 bucket for the resource and upload the already created ssh keys pair. and tag the autoscaling with the bucket name and file name to be downloaded and stroed in the server at the time of intilization.
note that the public ssh key has to be added in the repository # aws_complete_infrastructure_terraform
