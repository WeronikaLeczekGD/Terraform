# Terraform

notes from instruction: Everything should be inside Terraform configuration files, manual changes are not allowed.
Try to use terraform modules to segregate resources by its types (compute, network).

## [1. Create a temporary VM with metadata script installing HTTP server (Apache) with simple website inside.](#ad-1)

## [2. Create an image from temporary VM.](#ad-2)

## [3. Terraform should create a scale set of 3 instances (use predefined image as source image for VMs), including external load balancer with health checks, everything should be done via terraform tf/tfstate files.](#ad-3)

## [4. Every host should display server number/hostname to ensure that load balancer is working. User should be able to connect to the website in High Availability mode via external load balancer IP](#ad-4)

## [5. Add firewall for accessing external load balancer from limited IP addresses range and only for certain ports.](#ad-5)

## [6. Use Public Cloud storage service as backend for Terraform state.](#ad-6)


## Ad 1

### Create a temporary VM with metadata script installing HTTP server (Apache) with simple website inside.

```
resource "aws_instance" "temp" {
  ami           = "ami-0aa7d40eeae50c9a9"
  instance_type = "t2.micro"
  key_name      = "spring-project-EC2"

  user_data = <<-EOF
              #!/bin/bash
              yum update -y
              yum install -y httpd
              echo "<html><body><h1>Hello World</h1><p>Server $(hostname)<p></body></html>" > /var/www/html/index.html
              service httpd start
              chkconfig httpd on
              EOF

  tags = {
    Name = "ExampleAppServerInstance"
    Owner = "wleczek"
    Project = "2022_intership_wro"
  }
}
```

## Ad 2

### Creating an image from temporary VM:

```
resource "aws_ami_from_instance" "ami" {
  name               = "ami_from_instance"
  source_instance_id = aws_instance.temp.id
  tags = {
    Name = "ExampleAppServerInstance"
    Owner = "wleczek"
    Project = "2022_intership_wro"
  }
}
```

### My image:

![Screenshot 2023-02-13 at 20 55 20](https://user-images.githubusercontent.com/114099418/218561501-4e974b0b-ebc3-4076-8067-277c326f76f4.png)


## Ad 3

### To create 3 instances I used:
```
resource "aws_instance" "group_3" {
  count = 3
  ami      = aws_ami_from_instance.ami.id
  instance_type = "t2.micro"
  key_name      = "spring-project-EC2"

  tags = {
    Name = "ExampleAppServerInstance"
    Owner = "wleczek"
    Project = "2022_intership_wro"
  }
}
```
### My 3 created instances + 1 temporary:

![Screenshot 2023-02-13 at 20 57 56](https://user-images.githubusercontent.com/114099418/218561984-6c781120-70df-427a-bd46-2deffe30502a.png)

### Load balancer with health checks:

```
resource "aws_elb" "example" {
  name               = "example-load-balancer"
  internal           = false
  security_groups    = [aws_security_group.example.id]
  cross_zone_load_balancing = true
  availability_zones = ["us-east-1a"]

    listener {
        instance_port     = 80
        instance_protocol = "http"
        lb_port           = 80
        lb_protocol       = "http"
    }

    health_check {
    target              = "HTTP:80/"
    interval            = 30
    healthy_threshold   = 2
    unhealthy_threshold = 2
    timeout             = 3
  }
}

# associate the load balancer with instances
resource "aws_elb_attachment" "example" {
  count = 3
  elb      = aws_elb.example.id
  instance = aws_instance.group_3[count.index].id
}
```
### My load balancer:

![Screenshot 2023-02-13 at 21 01 12](https://user-images.githubusercontent.com/114099418/218562662-a0cf489d-064a-40fd-904b-9a6af1ee7ef7.png)

### Attatched instances:

![Screenshot 2023-02-13 at 21 03 57](https://user-images.githubusercontent.com/114099418/218563105-e6fbc460-25e4-46d9-8fd5-108b006d138f.png)

### Health checks:

![Screenshot 2023-02-13 at 21 04 15](https://user-images.githubusercontent.com/114099418/218563205-d5338efd-7439-4df8-952a-5b5d5f9e06a2.png)


## Ad 4

### Every host should display server number/hostname to ensure that load balancer is working. User should be able to connect to the website in High Availability mode via external load balancer IP.

![Screenshot 2023-02-13 at 21 12 43](https://user-images.githubusercontent.com/114099418/218564807-2e23be40-a8ac-4246-8073-fe69174f6b26.png)


## Ad 5

### Adding security groups for accessing external load balancer from limited IP addresses range and only for certain ports:

```
resource "aws_security_group" "example" {
  name        = "example-security-group"
  description = "Allow incoming HTTP traffic"

  ingress{
      from_port   = 80
      to_port     = 80
      protocol    = "tcp"
      cidr_blocks = ["10.0.0.0/8"]
    }

  egress{
      from_port   = 0
      to_port     = 0
      protocol    = "-1"
      cidr_blocks = ["0.0.0.0/0"]
    }
}
```

### My security group:

![Screenshot 2023-02-13 at 21 19 55](https://user-images.githubusercontent.com/114099418/218566225-5d845200-d0f3-4278-800b-90b7f4cd3188.png)


## Ad 6

### Using Public Cloud storage service as backend for Terraform state:

```
terraform {
  backend "s3" {
    bucket = "terraform-state-bucket-wleczek" 
    key    = "s3/terraform-state-bucket-wleczek"
    region = "us-east-1"
  }
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 4.16"
    }
  }

  required_version = ">= 1.2.0"
}

provider "aws" {
  region = "us-east-1"
}

data "aws_vpc" "default" {
  default = true
}
```
### My bucket:

![Screenshot 2023-02-13 at 21 23 20](https://user-images.githubusercontent.com/114099418/218566810-fb47d340-07eb-4d65-b1ce-ecb02d606e59.png)


