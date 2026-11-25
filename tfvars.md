```` hcl
region      = "us-east-1"
proyecto    = "eduandes"
aplicacion  = "biblioteca"
responsable = "alumno-duoc"
ambiente    = "dev" # <- unico cambio para pasar a "prod"

vpc_cidr = "10.0.0.0/16"

subnets = {
  publica-1 = { cidr = "10.0.1.0/24", az = "us-east-1a", publica = true }
  publica-2 = { cidr = "10.0.2.0/24", az = "us-east-1b", publica = true }
  privada-1 = { cidr = "10.0.11.0/24", az = "us-east-1a", publica = false }
  privada-2 = { cidr = "10.0.12.0/24", az = "us-east-1b", publica = false }
}

servidores = {
  biblioteca-a = { instance_type = "t3.medium", disco_gb = 30, subnet = "publica-1" }
  biblioteca-b = { instance_type = "t3.medium", disco_gb = 30, subnet = "publica-1" }
  biblioteca-c = { instance_type = "t3.large", disco_gb = 60, subnet = "publica-2" }
}

db_clase   = "db.t3.micro"
db_usuario = "admin"
# db_password NO va aqui:  $env:TF_VAR_db_password = "Biblioteca2026"
````
