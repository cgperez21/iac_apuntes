## main.tf 1
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
  region = var.region_aws
}

resource "random_string" "random" {
  length  = 16
  special = false
  upper   = false
}

locals {
  local_random = random_string.random.result
}

module "vpc" {
  source        = "./modulos/vpc"
  entorno       = local.local_random
  cidr_vpc      = var.cidr_vpc
  cidr_subredes = var.cidr_subredes
}

module "ec2" {
  source       = "./modulos/ec2"
  entorno      = local.local_random
  ids_subredes = module.vpc.subredes_ids
  id_sg        = module.vpc.id_sg
  cantidad     = var.cantidad_ec2
  ami_id       = var.ami_id
  nombre_llave = var.nombre_llave
}

module "nlb" {
  count          = var.cantidad_nlb
  source         = "./modulos/nlb"
  entorno        = "${local.local_random}-${count.index + 1}"
  id_vpc         = module.vpc.vpc_id
  ids_subredes   = module.vpc.subredes_ids
  ids_instancias = module.ec2.ids_instancias
}

module "s3" {
  source          = "./modulos/s3"
  entorno         = local.local_random
  nombres_buckets = var.nombres_buckets
}
```
## maint.tf 2
```` hcl
terraform {
  required_providers {
    aws = { source = "hashicorp/aws", version = "~> 5.0" }
  }
}
provider "aws" {
  region = var.region
}

data "aws_caller_identity" "actual" {}

data "aws_ami" "linux" {
  most_recent = true
  owners      = ["amazon"]
  filter {
    name   = "name"
    values = ["al2023-ami-2023.*-x86_64"]
  }
}

locals {
  es_prod  = var.ambiente == "prod"
  prefijo  = format("%s-%s-%s", var.proyecto, var.aplicacion, var.ambiente)
  tags     = { Proyecto = var.proyecto, Ambiente = var.ambiente, Responsable = var.responsable }
  publicas = { for k, v in var.subnets : k => v if v.publica }
  privadas = { for k, v in var.subnets : k => v if !v.publica }
}

# ---------- Red ----------
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  tags                 = merge(local.tags, { Name = "${local.prefijo}-vpc" })
}

resource "aws_subnet" "main" {
  for_each                = var.subnets
  vpc_id                  = aws_vpc.main.id
  cidr_block              = each.value.cidr
  availability_zone       = each.value.az
  map_public_ip_on_launch = each.value.publica
  tags                    = merge(local.tags, { Name = "${local.prefijo}-${each.key}" })
}

resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
  tags   = merge(local.tags, { Name = "${local.prefijo}-igw" })
}

resource "aws_route_table" "publica" {
  vpc_id = aws_vpc.main.id
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }
  tags = merge(local.tags, { Name = "${local.prefijo}-rt" })
}

resource "aws_route_table_association" "publica" {
  for_each       = local.publicas
  subnet_id      = aws_subnet.main[each.key].id
  route_table_id = aws_route_table.publica.id
}

# ---------- Security groups ----------
resource "aws_security_group" "servidores" {
  name   = "${local.prefijo}-sg-srv"
  vpc_id = aws_vpc.main.id
  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
  tags = merge(local.tags, { Name = "${local.prefijo}-sg-srv" })
}

resource "aws_security_group" "rds" {
  name   = "${local.prefijo}-sg-rds"
  vpc_id = aws_vpc.main.id
  ingress { # solo desde el SG de los servidores
    from_port       = 3306
    to_port         = 3306
    protocol        = "tcp"
    security_groups = [aws_security_group.servidores.id]
  }
  tags = merge(local.tags, { Name = "${local.prefijo}-sg-rds" })
}

# ---------- Servidores (1 bloque, 3 instancias) ----------
resource "aws_instance" "servidor" {
  for_each                    = var.servidores
  ami                         = data.aws_ami.linux.id
  instance_type               = each.value.instance_type
  subnet_id                   = aws_subnet.main[each.value.subnet].id
  vpc_security_group_ids      = [aws_security_group.servidores.id]
  associate_public_ip_address = true
  user_data = <<-EOT
    #!/bin/bash
    dnf install -y httpd mariadb105
    systemctl enable --now httpd
    echo "<h1>Biblioteca Digital - ${each.key} - ${var.ambiente}</h1>" > /var/www/html/index.html
  EOT
  root_block_device {
    volume_size = local.es_prod ? each.value.disco_gb * 2 : each.value.disco_gb # condicional 1
  }
  tags = merge(local.tags, { Name = "${local.prefijo}-${each.key}" })
}

# ---------- NLB ----------
resource "aws_lb" "nlb" {
  name               = "${local.prefijo}-nlb"
  load_balancer_type = "network"
  subnets            = [for k in keys(local.publicas) : aws_subnet.main[k].id]
  tags               = merge(local.tags, { Name = "${local.prefijo}-nlb" })
}

resource "aws_lb_target_group" "tg" {
  name     = "${local.prefijo}-tg"
  port     = 80
  protocol = "TCP"
  vpc_id   = aws_vpc.main.id
  tags     = merge(local.tags, { Name = "${local.prefijo}-tg" })
}

resource "aws_lb_listener" "tcp" {
  load_balancer_arn = aws_lb.nlb.arn
  port              = 80
  protocol          = "TCP"
  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.tg.arn
  }
  tags = merge(local.tags, { Name = "${local.prefijo}-listener" })
}

resource "aws_lb_target_group_attachment" "srv" {
  for_each         = aws_instance.servidor # ciclo sobre los servidores
  target_group_arn = aws_lb_target_group.tg.arn
  target_id        = each.value.id
  port             = 80
}

# ---------- RDS ----------
resource "aws_db_subnet_group" "db" {
  name       = "${local.prefijo}-dbsg"
  subnet_ids = [for k in keys(local.privadas) : aws_subnet.main[k].id]
  tags       = merge(local.tags, { Name = "${local.prefijo}-dbsg" })
}

resource "aws_db_instance" "db" {
  identifier             = "${local.prefijo}-mysql"
  engine                 = "mysql"
  instance_class         = var.db_clase
  allocated_storage      = 20
  username               = var.db_usuario
  password               = var.db_password
  db_subnet_group_name   = aws_db_subnet_group.db.name
  vpc_security_group_ids = [aws_security_group.rds.id]
  publicly_accessible    = false
  multi_az               = local.es_prod # condicional 2
  skip_final_snapshot    = true
  tags                   = merge(local.tags, { Name = "${local.prefijo}-mysql" })
}

# ---------- S3 ----------
resource "aws_s3_bucket" "material" {
  bucket        = lower(format("%s-%s-%s-%s", var.proyecto, var.aplicacion, var.ambiente, data.aws_caller_identity.actual.account_id))
  force_destroy = true
  tags          = merge(local.tags, { Name = "${local.prefijo}-bucket" })
}

resource "aws_s3_bucket_versioning" "material" {
  bucket = aws_s3_bucket.material.id
  versioning_configuration {
    status = local.es_prod ? "Enabled" : "Suspended" # condicional 3
  }
}
````
