## Terraform provider https://registry.terraform.io/providers/hashicorp/aws/latest/docs 
```hcl
terraform {
  required_version = ">= 1.4.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    random = {
      source  = "hashicorp/random"
      version = "~> 3.1"
    }
  }
}
provider "aws" {
  region = "us-east-1" ***
}
```
## Random https://registry.terraform.io/providers/hashicorp/random/latest/docs/resources/string
```` hcl
resource "random_string" "random" {
  length           = 16
  special          = false
  upper            = false  
}
````
## vpc provider https://registry.terraform.io/providers/hashicorp/aws/latest/docs https://registry.terraform.io/providers/hashicorp/aws/latest/docs/data-sources/availability_zones
````hcl
resource "aws_vpc" "**main**" {
  cidr_block = "10.0.0.0/16" ***
  tags = {
    Name = local.nombre
}
}

data "aws_availability_zones" "**available**" {
  state = "available"
}
```````
## subnet provider https://registry.terraform.io/providers/-/aws/latest/docs/resources/subnet
```` hcl
resource "aws_subnet" "**main**" {
  vpc_id     = aws_vpc.**main**.id
  cidr_block = "10.0.1.0/24" *****
  map_public_ip_on_launch = true
  availability_zone       = data.aws_availability_zones.available.names[count.index]
  tags = {
    Name = "Main" *****
  }
````
## IGW https://registry.terraform.io/providers/-/aws/latest/docs/resources/internet_gateway
```` hcl
resource "aws_internet_gateway" "gw" {
  vpc_id = aws_vpc.**main**.id
  tags = {
    Name = "main" ***
  }
}
````
## Route Table https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/route_table.html
```` hcl
resource "aws_route_table" "example" {
  vpc_id = aws_vpc.example.id
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.example.id
  }
  tags = {
    Name = "example"
  }
}
````
## Route Table asociacion https://registry.terraform.io/providers/-/aws/latest/docs/resources/route_table_association
```` hcl
resource "aws_route_table_association" "example" {
  subnet_id      = aws_subnet.example.id
  route_table_id = aws_route_table.example.id
}
````
## Grupo de seguridad https://registry.terraform.io/providers/-/aws/5.7.0/docs/resources/security_group
```` hcl
resource "aws_security_group" "allow_tls" {
  name        = "allow_tls"
  description = "Allow TLS inbound traffic"
  vpc_id      = aws_vpc.main.id
  ingress {
    description      = "TLS from VPC"
    from_port        = 443
    to_port          = 443
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
    Name = "allow_tls"
  }
}
````
## NLB http://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/lb
```` hcl
resource "aws_lb" "test" {
  name               = "test-lb-tf"
  internal           = false
  load_balancer_type = "network"
  subnets            = [for subnet in aws_subnet.public : subnet.id]
  tags = {
    Environment = "production"
  }
}
````
## Target group NLB https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/lb_target_group
```` hcl
resource "aws_lb_target_group" "test" {
  name     = "tf-example-lb-tg"
  port     = local.puerto
  protocol = "TCP"
  vpc_id   = aws_vpc.main.id
}
````
## Lister NLB https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/lb_listener
```` hcl
resource "aws_lb_listener" "front_end" {
  load_balancer_arn = aws_lb.front_end.arn
  port              = "443"
  protocol          = "HTTPS"
  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.front_end.arn
  }
}
````
## Target Group enlace https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/lb_target_group_attachment
```` hcl
resource "aws_lb_target_group_attachment" "test" {
  target_group_arn = aws_lb_target_group.test.arn
  target_id        = aws_instance.test.id
  port             = 80
}
````
## s3 https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/s3_bucket
````hcl
resource "aws_s3_bucket" "example" {
  bucket = "my-tf-test-bucket"
}
````
## s3 privado https://registry.terraform.io/providers/-/aws/latest/docs/resources/s3_bucket_public_access_block
```` hcl
resource "aws_s3_bucket_public_access_block" "example" {
  bucket = aws_s3_bucket.example.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
````
## EC2
```` hcl https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/instance
resource "aws_instance" "example" {
  ami           = #****"
  instance_type = "t3.micro"
  subnet_id              = ****
  key_name               = *****
  vpc_security_group_ids = ****

  tags = {
    Name = "HelloWorld"
  }
}
````
## local-exec https://developer.hashicorp.com/terraform/language/provisioners
```` hcl
  provisioner "local-exec" {
    command = "echo 'The server's IP address is ${self.id}'>>log.txt"
    interpreter = var.interprete_local
  }
}
````
## remote-exec https://developer.hashicorp.com/terraform/language/provisioners
```` hcl
  connection {
    type     = "ssh"
    user     = "root"
    private_key = var.root_password
    host     = var.host
  }

  provisioner "remote-exec" {
    inline = [
      "bootstrap-cluster.sh ${join(" ", aws_instance.cluster[*].private_ip)}",
    ]
  }
}
````




