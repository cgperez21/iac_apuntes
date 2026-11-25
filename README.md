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







| Archivo | Equivale a | Pregunta que responde |
|---|---|---|
| `variables.tf` | Un **formulario en blanco** | ¿Qué datos necesito que me entreguen? |
| `terraform.tfvars` | El **formulario llenado** | ¿Cuánto vale cada dato? |
| `main.tf` | Los **planos** | ¿Qué hay que construir? |
| `outputs.tf` | La **entrega de llaves** | ¿Qué quiero ver cuando termine? |

```
````
## 1. `variables.tf` — el formulario en blanco

### 1.1 Anatomía de una variable

```hcl
variable "region" { type = string }
```

| Parte | Qué es | Por qué |
|---|---|---|
| `variable` | Palabra reservada de Terraform | Indica "estoy declarando un dato de entrada" |
| `"region"` | El nombre | Así se usará después: `var.region` |
| `type = string` | El tipo de dato | Terraform rechaza valores del tipo equivocado antes de crear nada |

### 1.2 Los tipos

| Tipo | Se escribe en tfvars así | Para qué |
|---|---|---|
| `string` | `"dev"` | Textos: nombres, regiones, CIDR |
| `number` | `30` | Cantidades: GB de disco, puertos |
| `bool` | `true` / `false` | Sí/No |
| `list(string)` | `["a", "b"]` | Lista ordenada |
| `map(string)` | `{ a = "x", b = "y" }` | Pares clave → valor |
| `map(object({...}))` | ver abajo | **Una "tabla"**: cada fila es un elemento con varios campos |

### 1.3 `map(object(...))`

```hcl
variable "servidores" {
  type = map(object({
    instance_type = string
    disco_gb      = number
    subnet        = string
  }))
}
```

Es esto:

| clave (nombre de la fila) | instance_type | disco_gb | subnet |
|---|---|---|---|
| `biblioteca-a` | `t3.medium` | 30 | `publica-1` |
| `biblioteca-b` | `t3.medium` | 30 | `publica-1` |
| `biblioteca-c` | `t3.large` | 60 | `publica-2` |

- `map` = "tabla con filas identificadas por un nombre".
- `object({...})` = "las columnas que tiene cada fila y su tipo".

### 1.4 `validation`

```hcl
variable "ambiente" {
  type = string
  validation {
    condition     = contains(["dev", "prod"], var.ambiente)
    error_message = "ambiente debe ser dev o prod."
  }
}
```
- `contains(lista, valor)` → `true` si el valor está en la lista.
- Si da `false`, Terraform **se detiene** y muestra `error_message`

### 1.5 `sensitive = true`

```hcl
variable "db_password" {
  type      = string
  sensitive = true
}
```
- La contraseña **no** va en `terraform.tfvars`. Entra por `$env:TF_VAR_db_password`.

### 1.6 Qué significa cada variable

| Variable | Por qué existe | En otro contexto |
|---|---|---|
| `region` | Dónde se crea todo | Cambiar el valor (`us-west-2`, `sa-east-1`…) |
| `proyecto`, `aplicacion` | Nombres para recursos y bucket | Nombre del nuevo cliente/app |
| `responsable` | Tag de quién administra | Otro nombre |
| `ambiente` | Interruptor `dev`/`prod` | Agregar `"qa"` en la validación si hay 3 ambientes |
| `vpc_cidr` | Rango de IPs de la red | Cualquier rango privado (`192.168.0.0/16`) |
| `subnets` | Las subredes | Más/menos filas, otros CIDR y zonas |
| `servidores` | Los servidores | Más/menos filas, otros tipos y discos |
| `db_clase`, `db_usuario` | Tamaño y usuario de la BD | Otros valores |
| `db_password` | Contraseña | Siempre por `TF_VAR_db_password` |

---

## 2. `terraform.tfvars` — el formulario llenado

### Reglas de escritura

```hcl
region      = "us-east-1"     # texto  → entre comillas
disco_gb    = 30              # número → sin comillas
activo      = true            # bool   → sin comillas

subnets = {                   # mapa de objetos
  publica-1 = { cidr = "10.0.1.0/24", az = "us-east-1a", publica = true }
  privada-1 = { cidr = "10.0.11.0/24", az = "us-east-1a", publica = false }
}
```

1. **El nombre a la izquierda debe ser idéntico** al declarado en `variables.tf`.
> **Truco para cambiar de ambiente:** `terraform plan -var="ambiente=prod"` anula el valor del archivo solo por esa ejecución.

---

## 3. `main.tf` — los planos, bloque por bloque

### 3.1 Bloques de configuración 

| Bloque | Qué hace | Por qué se necesita |
|---|---|---|
| `terraform { required_providers { aws = {...} } }` | Declara qué **plugin** usar (`hashicorp/aws`) y qué versión (`~> 5.0` = cualquier 5.x) | Terraform no sabe hablar con AWS por sí solo: `terraform init` descarga este plugin |
| `provider "aws" { region = var.region }` | Configura el plugin: en qué región trabajar | Las credenciales **no** van aquí: las toma de `~/.aws/credentials` |
| `data "aws_caller_identity" "actual" {}` | **Consulta** el ID de la cuenta AWS (no crea nada) | Se agrega al nombre del bucket porque los nombres de S3 deben ser únicos en **todo el mundo** |
| `data "aws_ami" "linux" {...}` | **Busca** la imagen (AMI) de Amazon Linux más reciente | El ID de una AMI cambia por región y con el tiempo; buscarla evita escribirlo a mano |

**`data` vs `resource`:** `resource` = *crear*; `data` = *consultar algo que ya existe*.

### 3.2 `locals` — variables internas calculadas

```hcl
locals {
  es_prod  = var.ambiente == "prod"
  prefijo  = format("%s-%s-%s", var.proyecto, var.aplicacion, var.ambiente)
  tags     = { Proyecto = var.proyecto, Ambiente = var.ambiente, Responsable = var.responsable }
  publicas = { for k, v in var.subnets : k => v if v.publica }
  privadas = { for k, v in var.subnets : k => v if !v.publica }
}
```

| Local | Qué es | Por qué |
|---|---|---|
| `es_prod` | `true` si el ambiente es prod | Se calcula **una vez** y se reutiliza en los 3 condicionales. Evita repetir `var.ambiente == "prod"` |
| `prefijo` | `"eduandes-biblioteca-dev"` | Nombre base de todos los recursos, armado con `format` |
| `tags` | Mapa de etiquetas | Se reutiliza en todos los recursos con `merge` |
| `publicas` | Solo las subredes con `publica = true` | Se necesitan separadas: las públicas llevan NLB y ruta a internet; las privadas, la RDS |
| `privadas` | Solo las subredes con `publica = false` | Idem |

> "Para cada **clave k** y **valor v** del mapa `subnets`, arma un mapa nuevo `k => v`, **pero solo si** `v.publica` es verdadero."
`!` significa "no": `!v.publica` = "las que NO son públicas".

### 3.3 Red

| Recurso | Qué hace | Por qué |
|---|---|---|
| `aws_vpc.main` | Crea la red privada virtual con `cidr_block = var.vpc_cidr` | Es el contenedor de todo. `enable_dns_hostnames = true` permite resolver nombres dentro de la VPC (necesario para el endpoint de la RDS) |
| `aws_subnet.main` + `for_each = var.subnets` | Crea **una subred por cada fila** del mapa | Un bloque en vez de 4. `each.key` = nombre de la fila; `each.value.cidr`, `.az`, `.publica` = sus columnas |
| `map_public_ip_on_launch = each.value.publica` | Las instancias de subredes públicas reciben IP pública | Sin esto no se podría entrar por internet |
| `aws_internet_gateway.main` | La "puerta" de la VPC hacia internet | Sin ella nada sale ni entra |
| `aws_route_table.publica` (ruta `0.0.0.0/0` → gateway) | "Todo el tráfico a internet pasa por la puerta" | **Una subred es pública porque tiene esta ruta**, no por su nombre |
| `aws_route_table_association.publica` + `for_each = local.publicas` | Conecta la tabla de rutas **solo** con las subredes públicas | Las privadas no deben tener salida a internet |

### 3.4 Security groups

| Recurso | Reglas | Por qué |
|---|---|---|
| `aws_security_group.servidores` | **Entra** 80 (web) y 22 (SSH) desde cualquier IP; **sale** todo | Los usuarios llegan por 80; el administrador por 22 |
| `aws_security_group.rds` | **Entra** 3306 **solo desde** `security_groups = [servidores]` | En vez de una IP se nombra al **grupo de los servidores**: solo ellos pueden entrar. Nada desde internet ni desde su PC |

Campos de una regla: `from_port` / `to_port` (rango; igual para un solo puerto), `protocol = "tcp"` (`"-1"` = todos), `cidr_blocks` (IPs permitidas) o `security_groups` (grupos permitidos).

### 3.5 Servidores

```hcl
resource "aws_instance" "servidor" {
  for_each = var.servidores                      # 1 bloque → 3 servidores
  ami           = data.aws_ami.linux.id          # la imagen que buscamos
  instance_type = each.value.instance_type       # columna de la tabla
  subnet_id     = aws_subnet.main[each.value.subnet].id
  ...
}
```

| Línea | Explicación |
|---|---|
| `for_each = var.servidores` | Crea una instancia por cada fila del mapa |
| `each.value.instance_type` | Toma la columna `instance_type` de **esa** fila |
| `aws_subnet.main[each.value.subnet].id` | `each.value.subnet` vale `"publica-1"`. Se usa como **llave** para buscar esa subred dentro de las creadas con `for_each` y tomar su `id` |
| `vpc_security_group_ids = [...]` | Qué firewall usa (entre corchetes porque es una **lista**) |
| `associate_public_ip_address = true` | IP pública para poder verla desde internet |
| `user_data = <<-EOT ... EOT` | Script que corre al **arrancar**: instala Apache y el cliente MySQL. `${each.key}` inserta el nombre del servidor |
| `volume_size = local.es_prod ? disco * 2 : disco` | **Condicional 1.** En prod el disco se duplica |
| `tags = merge(local.tags, { Name = "..." })` | Tags comunes **+** un `Name` propio de cada servidor |

### 3.6 Balanceador (NLB): 4 piezas que van juntas

| Recurso | Qué es | Analogía |
|---|---|---|
| `aws_lb` (`load_balancer_type = "network"`) | El balanceador, en las subredes públicas | El recepcionista |
| `aws_lb_target_group` | La lista de servidores destino | La lista de empleados disponibles |
| `aws_lb_listener` | Puerto en que escucha y a qué grupo reenvía | La puerta de entrada |
| `aws_lb_target_group_attachment` + `for_each = aws_instance.servidor` | Agrega **cada** servidor a la lista | Anotar a cada empleado |

`subnets = [for k in keys(local.publicas) : aws_subnet.main[k].id]` →
"`keys()` entrega los nombres de las subredes públicas; el `for` busca cada una y toma su ID, y arma una lista".

### 3.7 RDS (base de datos)

| Recurso / línea | Qué hace | Por qué |
|---|---|---|
| `aws_db_subnet_group` | Agrupa las subredes **privadas** | RDS exige saber en qué subredes puede vivir (mínimo 2 zonas distintas) |
| `engine = "mysql"` | Motor de base de datos | Cambia a `"postgres"`, `"mariadb"`… |
| `instance_class = var.db_clase` | Tamaño | Variable |
| `allocated_storage = 20` | GB de disco | Mínimo permitido para MySQL |
| `username` / `password` | Credenciales de administrador | `password` viene de `TF_VAR_db_password` |
| `vpc_security_group_ids = [rds]` | El firewall restrictivo | Solo servidores |
| `publicly_accessible = false` | Sin IP pública | Requisito: no accesible desde internet |
| `multi_az = local.es_prod` | **Condicional 2.** `true` crea réplica de respaldo en otra zona | Alta disponibilidad solo en prod |
| `skip_final_snapshot = true` | No pide copia final al borrar | Si no, `terraform destroy` falla |

### 3.8 Bucket S3

| Recurso / línea | Qué hace | Por qué |
|---|---|---|
| `bucket = lower(format("%s-%s-%s-%s", ...))` | Nombre generado: proyecto-aplicación-ambiente-idCuenta | `format` arma el texto; `lower` evita mayúsculas (S3 las prohíbe) |
| `force_destroy = true` | Permite borrar el bucket aunque tenga archivos | Si no, `terraform destroy` falla con archivos subidos |
| `aws_s3_bucket_versioning` | Configura el versionado en un recurso aparte | En AWS provider 5.x el versionado ya no va dentro del bucket |
| `status = local.es_prod ? "Enabled" : "Suspended"` | **Condicional 3** | Historial de archivos solo en prod |

---

## 4. `outputs.tf` — lo que se muestra al final

```hcl
output "nlb_dns"       { value = aws_lb.nlb.dns_name }
output "rds_endpoint"  { value = aws_db_instance.db.endpoint }
output "bucket_name"   { value = aws_s3_bucket.material.bucket }
output "ips_servidores" { value = { for k, s in aws_instance.servidor : k => s.public_ip } }
```

- `output "nombre" { value = ... }` → después del `apply` se imprime. También con `terraform output`.
- La forma de `value`: **`tipo_recurso.nombre_local.atributo`**. Para saber qué atributos existen, mire la documentación del recurso (sección *Attribute Reference*).
- El último es una expresión `for`: "para cada servidor (clave k, objeto s), arma `nombre => IP pública`".
- No se puede imprimir un output que contenga un valor `sensitive` sin marcarlo `sensitive = true`.

---


## 5. cómo adaptar a cualquier enunciado (7 pasos)

1. **Subraye los sustantivos del enunciado** (VPC, subredes, servidores, balanceador, BD, bucket…). Cada uno es un recurso.
2. **Anote los datos que cambian** (nombres, tamaños, CIDR, cantidades, puertos). Cada uno es una variable.
3. **Decida el tipo de cada variable**: ¿un dato? → `string`/`number`/`bool`. ¿Varios elementos iguales con distintos datos? → `map(object)`.
4. **Escriba `variables.tf`** (solo nombre y tipo) y luego **`terraform.tfvars`** (valores).
5. **Escriba `main.tf` en este orden:** provider → data → locals → red → security groups → servidores → balanceador → BD → bucket.
6. **Aplique los condicionales** con `local.es_prod ? valor_prod : valor_dev`.
7. **Escriba `outputs.tf`** con lo que pide el enunciado y ejecute `init → validate → plan`.


| Si el enunciado dice… | Escriba… |
|---|---|
| "una VPC con N subredes" | `aws_vpc` + `aws_subnet` con `for_each` sobre un `map(object)` |
| "subredes públicas y privadas" | Campo `publica = bool` + locals con `for … if` + IGW + tabla de rutas |
| "servidores con distinto tamaño/disco" | `aws_instance` con `for_each` sobre `map(object({instance_type, disco_gb, subnet}))` |
| "balanceador con listener TCP/HTTP" | `aws_lb` + `aws_lb_target_group` + `aws_lb_listener` + attachment con `for_each` |
| "base de datos no accesible desde internet" | `publicly_accessible = false` + SG con `security_groups = [sg_servidores]` + subnet group privado |
| "contraseña protegida" | Variable `sensitive = true` sin default + `TF_VAR_...` |
| "bucket con nombre generado con format()" | `bucket = lower(format("%s-%s-%s", a, b, c))` |
| "en prod X, en dev Y" | `local.es_prod ? X : Y` |
| "tags comunes en todos los recursos" | `locals { tags = {...} }` + `tags = merge(local.tags, {...})` |
| "cambiar de ambiente solo con la variable" | Variable `ambiente` + `es_prod` + nada de valores fijos dependientes del ambiente |
| "agregar otro servidor sin tocar el código" | `for_each` + una fila nueva en `tfvars` |
| "usar al menos dos funciones" | `format`, `lower`, `merge`, `keys`, `length`, `join`, `contains`… |

---

## 7. EJEMPLO

**Enunciado ficticio:** *"TiendaAndes quiere una tienda online en AWS: región Oregon, red 192.168.0.0/16, 1 subred pública y 2 privadas, 2 servidores (web y api), base de datos PostgreSQL, bucket para imágenes y 3 ambientes: dev, qa y prod. En prod los discos se duplican y la BD es Multi-AZ."*

### 7.1 `variables.tf` 

```hcl
variable "ambiente" {
  type = string
  validation {
    condition     = contains(["dev", "qa", "prod"], var.ambiente)   # ← ahora 3 ambientes
    error_message = "ambiente debe ser dev, qa o prod."
  }
}

variable "db_motor"  { type = string }     # ← NUEVA: "postgres"
variable "db_puerto" { type = number }     # ← NUEVA: 5432
```

### 7.2 `terraform.tfvars`

```hcl
region      = "us-west-2"
proyecto    = "tiendaandes"
aplicacion  = "tienda"
responsable = "maria"
ambiente    = "qa"
vpc_cidr = "192.168.0.0/16"
subnets = {
  web-1   = { cidr = "192.168.1.0/24",  az = "us-west-2a", publica = true  }
  datos-1 = { cidr = "192.168.10.0/24", az = "us-west-2a", publica = false }
  datos-2 = { cidr = "192.168.20.0/24", az = "us-west-2b", publica = false }
}
servidores = {
  web = { instance_type = "t3.small",  disco_gb = 20, subnet = "web-1" }
  api = { instance_type = "t3.medium", disco_gb = 40, subnet = "web-1" }
}
db_clase   = "db.t3.micro"
db_usuario = "tienda_admin"
db_motor   = "postgres"
db_puerto  = 5432
```

### 7.3 `main.tf` 

```hcl
# (a) Los 3 ambientes: ya no basta un true/false para el disco. Se usa lookup() con un mapa:
locals {
  es_prod     = var.ambiente == "prod"                                   # sigue sirviendo para Multi-AZ y versionado
  factor_disco = lookup({ dev = 1, qa = 1, prod = 2 }, var.ambiente, 1)  # NUEVO
  # ... el resto de locals igual
}

# (b) En aws_instance:
root_block_device {
  volume_size = each.value.disco_gb * local.factor_disco    # antes: local.es_prod ? disco*2 : disco
}
user_data = <<-EOT
  #!/bin/bash
  dnf install -y httpd postgresql15                         # antes: mariadb105
  systemctl enable --now httpd
  echo "<h1>Tienda - ${each.key} - ${var.ambiente}</h1>" > /var/www/html/index.html
EOT

# (c) En aws_security_group.rds:
from_port = var.db_puerto                                   # antes: 3306
to_port   = var.db_puerto
# (d) En aws_db_instance.db:
engine = var.db_motor                                       # antes: "mysql"
```

### 7.4 Qué hacer si la BD fuera la misma pero el cliente pide 2 balanceadores, 5 servidores, etc.

- 5 servidores → 5 filas en `servidores`.
- Más subredes → más filas en `subnets`.
- Otro bucket → copiar el bloque `aws_s3_bucket` con otro nombre local, o usar `for_each = toset(["imagenes","backups"])` con `each.key` en el nombre.

---

## 8. 
| Cambio pedido | Dónde | Cómo |
|---|---|---|
| Agregar un servidor | `terraform.tfvars` | Nueva fila en `servidores` |
| Cambiar tipo de instancia | `terraform.tfvars` | Cambiar `instance_type` de la fila |
| Cambiar región | `terraform.tfvars` | `region = "..."` **y** las `az` de `subnets` |
| Cambiar rango de red | `terraform.tfvars` | `vpc_cidr` y los `cidr` de cada subred (deben estar **dentro** del `vpc_cidr`) |
| Agregar una subred | `terraform.tfvars` | Nueva fila en `subnets` |
| Disco ×3 en prod | `main.tf` | `each.value.disco_gb * 3` en el lado `prod` del ternario |
| Una instancia más grande solo en prod | `main.tf` | `instance_type = local.es_prod ? "t3.large" : each.value.instance_type` |
| RDS más grande en prod | `main.tf` | `instance_class = local.es_prod ? "db.t3.medium" : var.db_clase` |
| Otro puerto de la aplicación | `variables.tf` + `tfvars` + `main.tf` | Variable `puerto_app` y usarla en SG, target group, listener y attachment |
| Agregar tag extra | `main.tf` | Un campo nuevo en `local.tags` (se aplica a todos) |
| Nuevo output | `outputs.tf` | `output "x" { value = recurso.nombre.atributo }` |
| Crear recurso solo en prod | `main.tf` | `count = local.es_prod ? 1 : 0` (y referenciar con `[0]`) |
| Usar `count` en vez de `for_each` | `main.tf` | `count = 3` y `count.index` (menos recomendado) |
| Una variable con lista de zonas | `variables.tf` | `type = list(string)` y `var.zonas[0]` |
| Ocultar otro dato | `variables.tf` | `sensitive = true` |

---

## 9. Patrones 

**Condicional con número / texto / booleano**
```hcl
volume_size   = local.es_prod ? 100 : 50
instance_type = local.es_prod ? "t3.large" : "t3.micro"
multi_az      = local.es_prod
```

**Un recurso por cada fila de un mapa**
```hcl
resource "TIPO" "nombre" {
  for_each = var.mapa
  campo    = each.value.columna
  tags     = merge(local.tags, { Name = "${local.prefijo}-${each.key}" })
}
```

**Un recurso por cada elemento de una lista**
```hcl
for_each = toset(var.lista)
name     = each.key
```

**Referenciar un recurso creado con `for_each` desde otro**
```hcl
subnet_id = aws_subnet.main["publica-1"].id          # por clave fija
subnet_id = aws_subnet.main[each.value.subnet].id    # por clave que viene de una variable
```

**Lista de IDs filtrada**
```hcl
[for k in keys(local.publicas) : aws_subnet.main[k].id]
```

**Security group que solo acepta a otro security group**
```hcl
ingress {
  from_port       = 3306
  to_port         = 3306
  protocol        = "tcp"
  security_groups = [aws_security_group.servidores.id]
}
```

**Nombre único generado**
```hcl
bucket = lower(format("%s-%s-%s", var.proyecto, var.aplicacion, var.ambiente))
```


- [ ] `$env:TF_VAR_db_password` definido **en esta ventana**.

**Errores típicos**

| Error | Causa | Solución |
|---|---|---|
| `Reference to undeclared input variable` | Usó `var.x` sin declarar `x` | Declararla en `variables.tf` |
| `Unsupported argument` en tfvars | Nombre mal escrito en el tfvars | Igualar con `variables.tf` |
| `Invalid index` / `no matching … key` | `subnet = "publica1"` pero la clave es `"publica-1"` | Escribir la clave exacta |
| `InvalidSubnet` / zona inválida | `az` de otra región | Ajustar `az` a la `region` |
| `DBSubnetGroupDoesNotCoverEnoughAZs` | Subredes privadas en una sola zona | Dos subredes privadas en zonas distintas |
| `Inconsistent conditional result types` | Ramas del ternario de tipos distintos | Ambas ramas `string` o ambas `number` |
| `Invalid for_each argument` | `for_each` sobre una **lista** | Usar un mapa o `toset(lista)` |
| `CIDR … is not within VPC` | Subred fuera del rango de la VPC | Corregir `cidr` |
| Contraseña rechazada por RDS | Tiene `/`, `@`, `"` o espacio, o < 8 caracteres | Cambiarla |

