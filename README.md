## Comandos

```bash
terraform init        # descarga providers y registra módulos (Cada vez que agrego/cambio un "source" → "terraform init")
terraform fmt         # ordena/indenta el código
terraform validate    # revisa sintaxis y referencias
terraform plan        # muestra qué va a crear/cambiar/borrar
terraform apply -auto-approve    # crea la infraestructura
terraform output      # vuelve a mostrar los outputs
terraform destroy     # borra todo lo que creó
```

Símbolos: `+` crear, `-` destruir, `~` modificar, `-/+` destruir y recrear.

---

## Módulo raíz

| Bloque | Para qué | Cómo |
|---|---|---|
| `terraform { required_version = ">= 1.0.0" required_providers {` | Fija versiones de Terraform y providers | `aws, source= "hashicorp", version= "~> 5.0"`, `random, "3.1.0"`|
| `provider "aws"` | Dice en qué región y cuenta trabajar | `region = "us-east-1"` |
| `resource "random_string"` | Generar un sufijo | `length = 6`, `special = false`, `upper = false` |
| `locals` | Alias reutilizable | `local_random = random_string.random.result` |
| `module "..."` | Llama a un módulo hijo y le pasa inputs | `source = "./modulos/vpc"` + argumentos |

---

## Cómo viajan los datos
- Output = lo que un módulo entrega hacia afuera.
- Variable = lo que un módulo recibe desde afuera.
# Ejemplo de subredes para la ec2:
1. `modulos/vpc/outputs.tf` → `main.tf(raiz)` → `ec2/variables.tf` → `ec2/main.tf` 
2.  `subredes_ids` → `id_subred = module.vpc...`  → `id_subred`  → `subnet_id = var.id_subred`
3. `lo publica` → `lo transporta` → `lo recibe` → `lo usa` 

| Quiero usar | Escribo |
|---|---|
| Usar una variable dentro del módulo | `var.<nombre>` |
| Usar un local | `local.<nombre>` |
| Leer el output de un módulo (desde la raíz) | `module.<nombre>.<output>` |
| Leer un atributo de un recurso | `<tipo>.<nombre>.<atributo>` |
| Leer un data source | `data.<tipo>.<nombre>.<atributo>` |

---

### Count
- **Qué es:** Crea "X" copias de un recurso. Los recursos se identifican por posición, si borro el del medio, los siguientes se recrean.
- **Para qué:** no repetir bloques (2 subredes, 2 EC2, 2 buckets).
- **Cómo:** `"count = 2"`. Dentro del recurso, `{count.index+1}` vale 0, 1, 2…
  
```hcl
resource "aws_subnet" "subred_prueba" {
  count      = 2
  cidr_block = "10.0.${count.index+1}.0/24"
  tags       = { Name = "subred-${count.index+1}" }
}
```
- Acceder a uno: `aws_subnet.subred_prueba[0].id`
- Acceder a todos: `aws_subnet.subred_prueba[*].id`

### Variables
- **Qué es:** Entradas parametrizables de un módulo.
- **Para qué:** Reutilizar el mismo código con distintos valores.
- **Cómo:** se declaran en `variables.tf` y se usan con `var.`.

```hcl
variable "entorno"       { type = string } 
variable "ids_subredes"  { type = list(string) } 
variable "region_aws" {
  type    = string
  default = "us-east-1"                        
}
```
- Tipos: `string`, `number`, `bool`, `list(...)`, `map(...)`, `set(...)`, `object({...})`.
- Si una variable no tiene `default`, el módulo que la llama debe pasarla, si no → error `Missing required argument`.
- Formas de dar valor: en el bloque `module` `-var="x=1"`, archivo `terraform.tfvars`, o variable de entorno `TF_VAR_x`.

### Locals
- **Qué es:** Variable interna y de sólo lectura que sirve para asignar un nombre a una expresión.
- **Para qué:** Evitar repetir expresiones largas.
- **Cómo:** bloque `locals {}` y se usa como `local.nombre` (singular al usar, plural al declarar).

```hcl
locals {
  local_random = random_string.random.result
}
```

### Random
- **Qué es:** Provider que genera valores aleatorios
- **Para qué:** Evito colisiones, cómo en S3.
- **Cómo:** `random_string.random.result` entrega el texto. `upper = false` y `special = false`
* El valor se guarda en el state, no cambia en cada `apply`

### Local-exec
- **Qué es:** ejecuta un comando donde corro Terraform (en mi PC).
- **Para qué:** dejar logs, llamar scripts, notificar, etc.
- **Cómo:** `provisioner` dentro del recurso. `self` = el propio recurso.

```hcl
provisioner "local-exec" {
  command     = "echo 'Instancia ${self.id} desplegada' >> log_instancias.txt"
  interpreter = ["PowerShell", "-Command"]
}
```
- El archivo se crea en la carpeta desde donde ejecuto terraform apply.
- Solo corre al crear el recurso, no en cada apply. Si falla, el recurso queda tainted.

### Remote-exec
- **Qué:** ejecuta comandos dentro de la instancia por SSH.
- **Para qué:** instalar y configurar software 
- **Cómo:** `provisioner "remote-exec"` + bloque `connection`.

```hcl
provisioner "remote-exec" {
  inline = [
    "sudo yum install -y httpd",
    "sudo systemctl enable --now httpd"
  ]
  connection {
    type        = "ssh"
    user        = "ec2-user"                          
    private_key = file("${path.root}/labsuser.pem") 
    host        = self.public_ip
  }
}
```
**Requisitos para que funcione** :
1. Instancia con IP pública (`map_public_ip_on_launch = true`).
2. Subred con ruta a un Internet Gateway.
3. Security Group con puerto 22 abierto.
4. `key_name` coincide con la llave `.pem` usada.
5. Usuario correcto según AMI.

### Outputs
- **Qué es:** valores que un módulo "devuelve".
- **Para qué:** (1) pasar datos entre módulos, (2) mostrar datos al usuario al terminar.
- **Cómo:**

```hcl
output "dns_nlb" {
  value       = aws_lb.principal.dns_name
}
```
En la raíz para mostrarlo: `value = module.nlb.dns_nlb`.

### For_each
- **Qué es:** Es parecido a count, pero en vez de repetir un número de veces, recorre una colección, y cada copia queda identificada por un nombre, no por una posición.
- **Para qué:** Lo mismo que count, pero con for_each puedes darle a cada una su propio nombre, tipo, puerto, etc.
- **Cómo:** Dentro del recurso, `{each.key}` vale el nombre del elemento actual, y `{each.value}` vale el contenido del elemento actual

```hcl
variable "servidores" {
  type = map(string)
  default = {
    web = "t3.micro"
    app = "t3.small"
  }
}
resource "aws_instance" "ec2_prueba" {
  for_each      = var.servidores
  instance_type = each.value               
(...)
  tags = {
    Name = "ec2-${each.key}-${var.entorno}"  →  ec2-web-asbdjhbad, ec2-app-asdlkasndl
  }
}
```

$env:TF_LOG = "DEBUG" $env:TF_LOG_PATH = "catalina-perez.log"
