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
