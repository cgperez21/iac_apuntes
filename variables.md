## vpc variable.tf 1
```hcl
variable "region_aws" {
  description = "Región de AWS"
  type        = string
  default     = "us-east-1"
}
variable "cidr_vpc" {
  type    = string
  default = "10.0.0.0/16"
}
variable "cidr_subredes" {
  type    = list(string)
  default = ["10.0.1.0/24", "10.0.2.0/24"]
}
variable "cantidad_ec2" {
  type    = number
  default = 2
}
variable "ami_id" {
  type    = string
  default = "ami-04b938437b69a2c9c"
}
variable "nombre_llave" {
  type    = string
  default = "vockey"
}
variable "cantidad_nlb" {
  type    = number
  default = 2
}
variable "nombres_buckets" {
  type    = list(string)
  default = ["datos", "logs"]
}
```
## variables.tf 2
```` hcl
# Solo declaraciones (nombre + tipo). Los valores van en terraform.tfvars.

variable "region" { type = string }
variable "proyecto" { type = string }
variable "aplicacion" { type = string }
variable "responsable" { type = string }

variable "ambiente" {
  type = string
  validation {
    condition     = contains(["dev", "prod"], var.ambiente)
    error_message = "ambiente debe ser dev o prod."
  }
}

variable "vpc_cidr" {type = string }

variable "subnets" {
  type = map(object({
    cidr    = string
    az      = string
    publica = bool
  }))
}

variable "servidores" {
  type = map(object({
    instance_type = string
    disco_gb      = number
    subnet        = string
  }))
}

variable "db_clase" { type = string }
variable "db_usuario" { type = string }

variable "db_password" {
  type      = string
  sensitive = true # el valor llega por TF_VAR_db_password, no por tfvars
}
````
