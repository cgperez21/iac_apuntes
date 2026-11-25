## Texto

| Función | Para qué | Cómo se usa | Caso de ejemplo |
|---|---|---|---|
| `format` | Armar un texto con una plantilla (`%s` texto, `%d` número, `%02d` número con 2 dígitos) | `format("%s-%s", "a", "b")` → `"a-b"` | *"Nombre del bucket a partir de proyecto, aplicación y ambiente"* → `format("%s-%s-%s", var.proyecto, var.aplicacion, var.ambiente)` |
| `lower` / `upper` | Pasar a minúsculas / mayúsculas | `lower("Mi-Bucket")` → `"mi-bucket"` | *"S3 no acepta mayúsculas"* → `lower(format(...))` |
| `join` | Unir una lista en un solo texto | `join(", ", ["a","b"])` → `"a, b"` | *"Mostrar los nombres de los servidores en un output"* → `join(", ", keys(var.servidores))` |
| `split` | Dividir un texto en lista | `split("-", "us-east-1")` → `["us","east","1"]` | *"Obtener solo la región de un nombre"* → `split("-", var.zona)[0]` |
| `replace` | Reemplazar parte de un texto | `replace("mi app", " ", "-")` → `"mi-app"` | *"Nombre con espacios que debe ser válido en AWS"* |
| `substr` | Extraer un trozo (parte en 0) | `substr("abcdef", 2, 3)` → `"cde"` | *"Usar solo las 3 primeras letras del ambiente"* → `substr(var.ambiente, 0, 3)` |
| `trimspace` | Quitar espacios al inicio y al final | `trimspace("  hola ")` → `"hola"` | *"Limpiar un valor ingresado por el usuario"* |

## Colecciones (listas y mapas)

| Función | Para qué | Cómo se usa | Caso de ejemplo |
|---|---|---|---|
| `length` | Contar elementos de lista, mapa o texto | `length(["a","b"])` → `2` | *"Cuántos servidores hay"* → `length(var.servidores)` |
| `merge` | Unir mapas (si se repite una clave, **gana el último**) | `merge({a=1}, {b=2})` → `{a=1, b=2}` | *"Agregar el tag Name a los tags comunes"* → `merge(local.tags, { Name = "web" })` |
| `lookup` | Buscar una clave en un mapa, con valor por defecto | `lookup(mapa, "clave", "defecto")` | *"Tamaño de instancia según ambiente"* → `lookup({dev="t3.micro", prod="t3.large"}, var.ambiente, "t3.micro")` |
| `keys` | Lista de claves de un mapa | `keys({a=1, b=2})` → `["a","b"]` | *"Obtener los nombres de las subredes públicas"* → `keys(local.subnets_publicas)` |
| `values` | Lista de valores de un mapa | `values({a=1, b=2})` → `[1,2]` | *"Recorrer solo los valores de un mapa"* |
| `toset` | Convertir lista en set (sin repetidos ni orden) | `toset(["a","a","b"])` → `["a","b"]` | *"Crear un bucket por cada nombre de una lista"* → `for_each = toset(["logs","datos"])` |
| `tolist` | Convertir set en lista | `tolist(toset(["a","b"]))` | *"Acceder a un elemento por índice de un set"* |
| `contains` | ¿Un valor está en la lista? (`true`/`false`) | `contains(["dev","prod"], "qa")` → `false` | *"Validar que ambiente sea dev o prod"* → `condition = contains(["dev","prod"], var.ambiente)` |
| `element` | Elemento por índice, con vuelta circular | `element(["a","b","c"], 4)` → `"b"` | *"Repartir 5 servidores entre 2 zonas"* → `element(var.zonas, count.index)` |
| `concat` | Unir listas | `concat([1,2], [3])` → `[1,2,3]` | *"Juntar subredes públicas y privadas en una sola lista"* |
| `flatten` | Aplanar listas anidadas | `flatten([[1,2],[3]])` → `[1,2,3]` | *"Lista de IDs que viene en grupos"* |
| `distinct` | Quitar repetidos de una lista | `distinct([1,1,2])` → `[1,2]` | *"Zonas repetidas en una lista"* |

## Archivos y plantillas

| Función | Para qué | Cómo se usa | Caso de ejemplo |
|---|---|---|---|
| `file` | Leer un archivo tal cual | `file("${path.module}/clave.pub")` | *"Cargar una llave SSH pública"* |
| `templatefile` | Leer una plantilla y reemplazar `${variables}` | `templatefile("userdata.tpl", { ambiente = var.ambiente })` | *"Script de arranque distinto por servidor"* → `user_data` de una EC2 |

## Red

| Función | Para qué | Cómo se usa | Caso de ejemplo |
|---|---|---|---|
| `cidrsubnet` | Calcular una subred dentro de una red mayor | `cidrsubnet("10.0.0.0/16", 8, 1)` → `"10.0.1.0/24"` | *"Generar los CIDR de las 4 subredes sin escribirlos a mano"* |

## Conversión y utilidad

| Función | Para qué | Cómo se usa | Caso de ejemplo |
|---|---|---|---|
| `tostring` / `tonumber` / `tobool` | Cambiar el tipo de un valor | `tonumber("3.14")` → `3.14` | *"Un valor llega como texto y se necesita número"* |
| `max` / `min` | Mayor / menor de varios números | `max(30, 60)` → `60` | *"Usar como disco el mayor entre el mínimo y el pedido"* |
| `try` | Usar un valor alternativo si la expresión falla | `try(var.config.puerto, 80)` | *"Campo opcional que podría no existir"* |
| `coalesce` | Primer valor que no sea vacío/`null` | `coalesce(var.nombre, "sin-nombre")` | *"Nombre por defecto si el usuario no lo entrega"* |
| `jsonencode` | Convertir un objeto de Terraform a JSON | `jsonencode({a = 1})` → `{"a":1}` | *"Política de un bucket S3 o rol IAM"* |
| `timestamp` | Fecha y hora actuales | `timestamp()` | *"Tag con la fecha de creación"* |

> **No son funciones, pero se preguntan junto con ellas:** el operador condicional `cond ? si_true : si_false`,
> la expresión `for` (`[for x in lista : upper(x)]`) y el splat `[*]`.
> Para probar cualquier ejemplo sin crear nada: `terraform console`.
