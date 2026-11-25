# Terraform — Biblioteca Digital

La práctica utiliza tres servidores Linux: Biblioteca A y B (`t3.medium`, disco de 30 GiB) y Biblioteca C (`t3.large`, disco de 60 GiB). El NLB y las EC2 usan las subredes públicas; RDS MySQL usa las privadas, y S3 no pertenece a ninguna subred.

En `dev` los discos mantienen esos tamaños, RDS es Single-AZ y S3 no tiene versionado habilitado. Con `prod`, los discos se duplican, RDS pasa a Multi-AZ y se habilita el versionado de S3. **El cambio a `prod` se revisa solamente con `terraform plan`**, porque algunos nombres dependen del ambiente y el plan puede incluir reemplazos.

Los ejemplos de código de abajo corresponden a **un único proyecto**. Las rutas indican dónde guardar cada archivo; no hay que juntar archivos de módulos distintos en una misma carpeta.

## Carpetas

```text
Biblioteca_Digital_Terraform_Simplificado/
  provider.tf
  variables.tf
  terraform.tfvars
  main.tf
  seguridad.tf
  outputs.tf
  .gitignore
  modulos/
    ec2/   main.tf, variables.tf, outputs.tf
    vpc/   main.tf, variables.tf, outputs.tf
    nlb/   main.tf, variables.tf, outputs.tf
    rds/   main.tf, variables.tf, outputs.tf
    s3/    main.tf, variables.tf, outputs.tf
```

`main.tf` raíz reúne los módulos. Los archivos de cada hijo contienen los recursos del servicio. Terraform lee todos los `.tf` de una misma carpeta en conjunto.

## EC2

Los tres servidores están guardados en la variable `servidores`, que es un `map(object(...))`. Cada entrada trae el tipo de instancia, tamaño del disco y subred. `for_each` recorre esas entradas sin duplicar `aws_instance`.

La subred no se escribe como un ID fijo: `lookup()` busca la clave `publica1` o `publica2` en el mapa que devuelve VPC. En `prod`, la condición `var.ambiente == "prod" ? 2 : 1` multiplica el tamaño base del disco por dos. `user_data` instala Apache al iniciar Linux y deja una página sencilla para comprobar el NLB; no representa la aplicación real. `<<-USERDATA` solo delimita el texto del script que se ejecutará en la EC2.

**`modulos/ec2/variables.tf`**

```hcl
variable "nombre" { type = string }
variable "ambiente" { type = string }
variable "ami_id" { type = string }
variable "servidores" {
  type = map(object({
    instance_type = string
    disk_size     = number
    subnet        = string
  }))
}
variable "subredes_publicas" { type = map(string) }
variable "security_group_id" { type = string }
variable "key_name" { type = string }
variable "tags" { type = map(string) }
```

**`modulos/ec2/main.tf`**

```hcl
resource "aws_instance" "servidores" {
  for_each                    = var.servidores
  ami                         = var.ami_id
  instance_type               = each.value.instance_type
  subnet_id                   = lookup(var.subredes_publicas, each.value.subnet)
  vpc_security_group_ids      = [var.security_group_id]
  associate_public_ip_address = true
  key_name                    = var.key_name

  root_block_device {
    volume_type = "gp3"
    volume_size = each.value.disk_size * (var.ambiente == "prod" ? 2 : 1)
    encrypted   = true
  }

  user_data = <<-USERDATA
    #!/bin/bash
    dnf install -y httpd
    systemctl enable --now httpd
    echo "<h1>Biblioteca Digital - ${each.key}</h1>" > /var/www/html/index.html
  USERDATA

  tags = merge(var.tags, { Name = "${var.nombre}-${each.key}" })
}
```

**`modulos/ec2/outputs.tf`**

```hcl
output "ids" {
  value = { for k, ec2 in aws_instance.servidores : k => ec2.id }
}
output "ips_publicas" {
  value = { for k, ec2 in aws_instance.servidores : k => ec2.public_ip }
}
```

El output `ids` devuelve un mapa con los identificadores de las EC2; NLB usa ese resultado para registrar los destinos. Las IP públicas se exponen para la conexión SSH de laboratorio.

## VPC

La red `10.0.0.0/16` contiene dos subredes públicas y dos privadas. Los mapas de entrada indican el CIDR y la zona de cada subred; `for_each` evita repetir cuatro bloques `aws_subnet`. Las subredes públicas se asocian a una tabla con ruta `0.0.0.0/0` al Internet Gateway. Las privadas tienen otra tabla sin salida directa a Internet.

Los IDs de las subredes públicas se entregan a EC2 y NLB. Los de las privadas se entregan a RDS. La tabla de rutas no asigna por sí sola IP pública a una EC2.

**`modulos/vpc/variables.tf`**

```hcl
variable "nombre" { type = string }
variable "cidr_vpc" { type = string }
variable "subredes_publicas" {
  type = map(object({ cidr = string, az = string }))
}
variable "subredes_privadas" {
  type = map(object({ cidr = string, az = string }))
}
variable "tags" { type = map(string) }
```

**`modulos/vpc/main.tf`**

```hcl
resource "aws_vpc" "principal" {
  cidr_block           = var.cidr_vpc
  enable_dns_support   = true
  enable_dns_hostnames = true
  tags                 = merge(var.tags, { Name = "vpc-${var.nombre}" })
}

resource "aws_subnet" "publicas" {
  for_each                = var.subredes_publicas
  vpc_id                  = aws_vpc.principal.id
  cidr_block              = each.value.cidr
  availability_zone       = each.value.az
  map_public_ip_on_launch = true
  tags                    = merge(var.tags, { Name = "${var.nombre}-${each.key}" })
}

resource "aws_subnet" "privadas" {
  for_each                = var.subredes_privadas
  vpc_id                  = aws_vpc.principal.id
  cidr_block              = each.value.cidr
  availability_zone       = each.value.az
  map_public_ip_on_launch = false
  tags                    = merge(var.tags, { Name = "${var.nombre}-${each.key}" })
}

resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.principal.id
  tags   = merge(var.tags, { Name = "igw-${var.nombre}" })
}

resource "aws_route_table" "publica" {
  vpc_id = aws_vpc.principal.id
  tags   = merge(var.tags, { Name = "rt-publica-${var.nombre}" })
}

resource "aws_route" "internet" {
  route_table_id         = aws_route_table.publica.id
  destination_cidr_block = "0.0.0.0/0"
  gateway_id             = aws_internet_gateway.igw.id
}

resource "aws_route_table_association" "publicas" {
  for_each       = aws_subnet.publicas
  subnet_id      = each.value.id
  route_table_id = aws_route_table.publica.id
}

resource "aws_route_table" "privada" {
  vpc_id = aws_vpc.principal.id
  tags   = merge(var.tags, { Name = "rt-privada-${var.nombre}" })
}

resource "aws_route_table_association" "privadas" {
  for_each       = aws_subnet.privadas
  subnet_id      = each.value.id
  route_table_id = aws_route_table.privada.id
}
```

**`modulos/vpc/outputs.tf`**

```hcl
output "id_vpc" {
  value = aws_vpc.principal.id
}
output "subredes_publicas_ids" {
  value = { for k, s in aws_subnet.publicas : k => s.id }
}
output "subredes_privadas_ids" {
  value = { for k, s in aws_subnet.privadas : k => s.id }
}
```

## NLB

El Network Load Balancer recibe conexiones TCP en el puerto 80 y las envía al target group, también en TCP 80. El listener conecta ambos recursos. `aws_lb_target_group_attachment` registra las tres EC2 recorriendo el mapa de IDs que entrega el módulo EC2. El control de salud TCP comprueba la respuesta del puerto, no el contenido de una página web.

El NLB utiliza las dos subredes públicas, el ID de VPC y su Security Group. Su DNS se publica como output para comprobar el acceso.

**`modulos/nlb/variables.tf`**

```hcl
variable "nombre" { type = string }
variable "vpc_id" { type = string }
variable "subredes_publicas" { type = list(string) }
variable "security_group_id" { type = string }
variable "servidores_ids" { type = map(string) }
variable "tags" { type = map(string) }
```

**`modulos/nlb/main.tf`**

```hcl
resource "aws_lb" "nlb" {
  name               = "nlb-${var.nombre}"
  internal           = false
  load_balancer_type = "network"
  subnets            = var.subredes_publicas
  security_groups    = [var.security_group_id]
  tags               = merge(var.tags, { Name = "nlb-${var.nombre}" })
}

resource "aws_lb_target_group" "tg" {
  name        = "tg-${var.nombre}"
  port        = 80
  protocol    = "TCP"
  target_type = "instance"
  vpc_id      = var.vpc_id
  tags = merge(var.tags, { Name = "tg-${var.nombre}" })
}

resource "aws_lb_listener" "tcp" {
  load_balancer_arn = aws_lb.nlb.arn
  port              = 80
  protocol          = "TCP"
  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.tg.arn
  }
  tags = merge(var.tags, { Name = "listener-${var.nombre}" })
}

resource "aws_lb_target_group_attachment" "registrados" {
  for_each         = var.servidores_ids
  target_group_arn = aws_lb_target_group.tg.arn
  target_id        = each.value
  port             = 80
}
```

**`modulos/nlb/outputs.tf`**

```hcl
output "dns" {
  value = aws_lb.nlb.dns_name
}
output "target_group_arn" {
  value = aws_lb_target_group.tg.arn
}
```

## RDS MySQL

RDS recibe dos subredes privadas mediante un DB Subnet Group. `publicly_accessible = false` impide publicar un endpoint accesible directamente desde Internet; además, el Security Group de RDS permite el puerto 3306 únicamente desde el Security Group de EC2.

`multi_az = var.ambiente == "prod"` cambia el despliegue según el ambiente. Multi-AZ mantiene una instancia de reserva para disponibilidad; no equivale a instalar una segunda base para consultar datos. `skip_final_snapshot = true` se conserva para poder destruir el laboratorio sin exigir una instantánea final, por lo que no debe trasladarse sin revisar a un entorno con datos reales.

**`modulos/rds/variables.tf`**

```hcl
variable "nombre" { type = string }
variable "ambiente" { type = string }
variable "subredes_privadas" { type = list(string) }
variable "security_group_id" { type = string }
variable "usuario" { type = string }
variable "nombre_bd" { type = string }
variable "password" {
  type      = string
  sensitive = true
}
variable "tipo" { type = string }
variable "disco_gb" { type = number }
variable "tags" { type = map(string) }
```

**`modulos/rds/main.tf`**

```hcl
resource "aws_db_subnet_group" "privadas" {
  name       = "dbsubnet-${var.nombre}"
  subnet_ids = var.subredes_privadas
  tags       = merge(var.tags, { Name = "dbsubnet-${var.nombre}" })
}

resource "aws_db_instance" "mysql" {
  identifier             = "rds-${var.nombre}"
  engine                 = "mysql"
  instance_class         = var.tipo
  allocated_storage      = var.disco_gb
  storage_type           = "gp3"
  storage_encrypted      = true
  db_name                = var.nombre_bd
  username               = var.usuario
  password               = var.password
  port                   = 3306
  multi_az               = var.ambiente == "prod"
  db_subnet_group_name   = aws_db_subnet_group.privadas.name
  vpc_security_group_ids = [var.security_group_id]
  publicly_accessible    = false

  skip_final_snapshot = true
  deletion_protection = false

  tags = merge(var.tags, { Name = "rds-${var.nombre}" })
}
```

**`modulos/rds/outputs.tf`**

```hcl
output "endpoint" {
  value = aws_db_instance.mysql.endpoint
}
output "direccion" {
  value = aws_db_instance.mysql.address
}
```

## S3

El bucket tiene un nombre único formado por el nombre base del proyecto y un sufijo de Random. En `format("%s-%s", var.nombre, var.sufijo)`, **cada `%s` representa un texto** y el guion queda entre ambos. Es equivalente a construir el nombre con interpolación, pero el ejercicio exige usar `format()`.

El acceso público queda bloqueado. La configuración de versionado solo se crea en `prod`. `force_destroy = true` permite borrar el bucket con sus objetos al terminar la práctica; no es una opción adecuada para un bucket que deba conservar archivos.

**`modulos/s3/variables.tf`**

```hcl
variable "nombre" { type = string }
variable "ambiente" { type = string }
variable "sufijo" { type = string }
variable "tags" { type = map(string) }
```

**`modulos/s3/main.tf`**

```hcl
resource "aws_s3_bucket" "material" {
  bucket        = format("%s-%s", var.nombre, var.sufijo)
  force_destroy = true
  tags          = merge(var.tags, { Name = "s3-${var.nombre}" })
}

resource "aws_s3_bucket_public_access_block" "privado" {
  bucket                  = aws_s3_bucket.material.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

resource "aws_s3_bucket_versioning" "produccion" {
  count  = var.ambiente == "prod" ? 1 : 0
  bucket = aws_s3_bucket.material.id

  versioning_configuration {
    status = "Enabled"
  }
}
```

**`modulos/s3/outputs.tf`**

```hcl
output "nombre_bucket" {
  value = aws_s3_bucket.material.bucket
}
```

## Archivos de la raíz

`provider.tf` selecciona AWS y Random. La AMI de Amazon Linux 2023 se consulta mediante `data "aws_ami"` para no depender de un ID copiado de otro laboratorio. `variables.tf` declara valores de entrada; `terraform.tfvars` contiene los datos concretos. En `main.tf` se pasan los outputs de VPC a los demás módulos y los IDs de EC2 al NLB. `values()` toma los IDs de un mapa de subredes y los entrega como lista a NLB y RDS.

`locals` prepara un nombre base y las etiquetas `proyecto`, `ambiente` y `responsable`. `merge()` añade la etiqueta `Name` particular de cada recurso. `seguridad.tf` mantiene las tres reglas de acceso esenciales: Internet al NLB por 80, NLB a EC2 por 80 y EC2 a RDS por 3306; SSH solo desde la IP autorizada.

**`provider.tf`**

```hcl
terraform {
  required_version = ">= 1.5.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"
    }
    random = {
      source  = "hashicorp/random"
      version = "~> 3.0"
    }
  }
}

provider "aws" {
  region = var.region
}
```

**`variables.tf`**

```hcl
variable "region" {
  type    = string
  default = "us-east-1"
}

variable "proyecto" {
  type    = string
  default = "eduandes"
}

variable "aplicacion" {
  type    = string
  default = "biblioteca"
}

variable "ambiente" {
  type    = string
  default = "dev"
  validation {
    condition     = contains(["dev", "prod"], var.ambiente)
    error_message = "ambiente solo puede ser dev o prod."
  }
}

variable "responsable" {
  type    = string
  default = "infraestructura"
}

variable "cidr_vpc" {
  type    = string
  default = "10.0.0.0/16"
}

variable "subredes_publicas" {
  type = map(object({
    cidr = string
    az   = string
  }))
  default = {
    publica1 = { cidr = "10.0.1.0/24", az = "us-east-1a" }
    publica2 = { cidr = "10.0.2.0/24", az = "us-east-1b" }
  }
}

variable "subredes_privadas" {
  type = map(object({
    cidr = string
    az   = string
  }))
  default = {
    privada1 = { cidr = "10.0.11.0/24", az = "us-east-1a" }
    privada2 = { cidr = "10.0.12.0/24", az = "us-east-1b" }
  }
}

variable "servidores" {
  type = map(object({
    instance_type = string
    disk_size     = number
    subnet        = string
  }))
}

variable "key_name" {
  type        = string
  description = "Nombre del Key Pair existente en EC2 (no la ruta al .pem)."
}

variable "ssh_cidr" {
  type        = string
  description = "IP publica del equipo autorizada para SSH, formato x.x.x.x/32."
}

variable "db_usuario" {
  type    = string
  default = "admin"
}

variable "db_nombre" {
  type    = string
  default = "biblioteca"
}

variable "db_password" {
  type      = string
  sensitive = true

  validation {
    condition     = length(var.db_password) >= 8 && length(var.db_password) <= 41
    error_message = "La contraseña de MySQL debe tener entre 8 y 41 caracteres."
  }
}

variable "rds_tipo" {
  type    = string
  default = "db.t3.micro"
}

variable "rds_disco_gb" {
  type    = number
  default = 20
}
```

**`terraform.tfvars`**

```hcl
region      = "us-east-1"
proyecto    = "eduandes"
aplicacion  = "biblioteca"
ambiente    = "dev"
responsable = "infraestructura"

key_name = "REEMPLAZAR_KEY_PAIR"
ssh_cidr = "203.0.113.10/32"

servidores = {
  biblioteca-a = {
    instance_type = "t3.medium"
    disk_size     = 30
    subnet        = "publica1"
  }
  biblioteca-b = {
    instance_type = "t3.medium"
    disk_size     = 30
    subnet        = "publica1"
  }
  biblioteca-c = {
    instance_type = "t3.large"
    disk_size     = 60
    subnet        = "publica2"
  }
}
```

**`main.tf`**

```hcl
resource "random_string" "sufijo" {
  length  = 6
  upper   = false
  lower   = true
  numeric = true
  special = false
}

data "aws_ami" "linux" {
  most_recent = true
  owners      = ["amazon"]
  filter {
    name   = "name"
    values = ["al2023-ami-2023.*-x86_64"]
  }
  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}

locals {
  nombre_base = "${var.proyecto}-${var.aplicacion}-${var.ambiente}"
  tags_comunes = {
    proyecto    = var.proyecto
    ambiente    = var.ambiente
    responsable = var.responsable
  }
}

module "vpc" {
  source            = "./modulos/vpc"
  nombre            = local.nombre_base
  cidr_vpc          = var.cidr_vpc
  subredes_publicas = var.subredes_publicas
  subredes_privadas = var.subredes_privadas
  tags              = local.tags_comunes
}

module "ec2" {
  source            = "./modulos/ec2"
  nombre            = local.nombre_base
  ambiente          = var.ambiente
  ami_id            = data.aws_ami.linux.id
  servidores        = var.servidores
  subredes_publicas = module.vpc.subredes_publicas_ids
  security_group_id = aws_security_group.ec2.id
  key_name          = var.key_name
  tags              = local.tags_comunes

  depends_on = [module.vpc, aws_vpc_security_group_egress_rule.ec2_salida]
}

module "nlb" {
  source            = "./modulos/nlb"
  nombre            = local.nombre_base
  vpc_id            = module.vpc.id_vpc
  subredes_publicas = values(module.vpc.subredes_publicas_ids)
  security_group_id = aws_security_group.nlb.id
  servidores_ids    = module.ec2.ids
  tags              = local.tags_comunes
}

module "rds" {
  source            = "./modulos/rds"
  nombre            = local.nombre_base
  ambiente          = var.ambiente
  subredes_privadas = values(module.vpc.subredes_privadas_ids)
  security_group_id = aws_security_group.rds.id
  usuario           = var.db_usuario
  nombre_bd         = var.db_nombre
  password          = var.db_password
  tipo              = var.rds_tipo
  disco_gb          = var.rds_disco_gb
  tags              = local.tags_comunes
}

module "s3" {
  source   = "./modulos/s3"
  nombre   = local.nombre_base
  ambiente = var.ambiente
  sufijo   = random_string.sufijo.result
  tags     = local.tags_comunes
}
```

**`seguridad.tf`**

```hcl
resource "aws_security_group" "nlb" {
  name_prefix = "sg-nlb-${local.nombre_base}-"
  description = "NLB publico de Biblioteca Digital"
  vpc_id      = module.vpc.id_vpc
  tags        = merge(local.tags_comunes, { Name = "sg-nlb-${local.nombre_base}" })
}

resource "aws_vpc_security_group_ingress_rule" "nlb_http" {
  security_group_id = aws_security_group.nlb.id
  ip_protocol       = "tcp"
  from_port         = 80
  to_port           = 80
  cidr_ipv4         = "0.0.0.0/0"
  tags              = merge(local.tags_comunes, { Name = "regla-nlb-http" })
}

resource "aws_vpc_security_group_egress_rule" "nlb_ec2" {
  security_group_id            = aws_security_group.nlb.id
  ip_protocol                  = "tcp"
  from_port                    = 80
  to_port                      = 80
  referenced_security_group_id = aws_security_group.ec2.id
  tags              = merge(local.tags_comunes, { Name = "regla-nlb-ec2" })
}

resource "aws_security_group" "ec2" {
  name_prefix = "sg-ec2-${local.nombre_base}-"
  description = "Instancias de Biblioteca Digital"
  vpc_id      = module.vpc.id_vpc
  tags        = merge(local.tags_comunes, { Name = "sg-ec2-${local.nombre_base}" })
}

resource "aws_vpc_security_group_ingress_rule" "ec2_http" {
  security_group_id            = aws_security_group.ec2.id
  ip_protocol                  = "tcp"
  from_port                    = 80
  to_port                      = 80
  referenced_security_group_id = aws_security_group.nlb.id
  tags              = merge(local.tags_comunes, { Name = "regla-ec2-http" })
}

resource "aws_vpc_security_group_ingress_rule" "ec2_ssh" {
  security_group_id = aws_security_group.ec2.id
  ip_protocol       = "tcp"
  from_port         = 22
  to_port           = 22
  cidr_ipv4         = var.ssh_cidr
  tags              = merge(local.tags_comunes, { Name = "regla-ec2-ssh" })
}

resource "aws_vpc_security_group_egress_rule" "ec2_salida" {
  security_group_id = aws_security_group.ec2.id
  ip_protocol       = "-1"
  cidr_ipv4         = "0.0.0.0/0"
  tags              = merge(local.tags_comunes, { Name = "regla-ec2-salida" })
}

resource "aws_security_group" "rds" {
  name_prefix = "sg-rds-${local.nombre_base}-"
  description = "MySQL privado; acceso exclusivo desde EC2"
  vpc_id      = module.vpc.id_vpc
  tags        = merge(local.tags_comunes, { Name = "sg-rds-${local.nombre_base}" })
}

resource "aws_vpc_security_group_ingress_rule" "rds_mysql" {
  security_group_id            = aws_security_group.rds.id
  ip_protocol                  = "tcp"
  from_port                    = 3306
  to_port                      = 3306
  referenced_security_group_id = aws_security_group.ec2.id
  tags              = merge(local.tags_comunes, { Name = "regla-rds-mysql" })
}
```

**`outputs.tf`**

```hcl
output "nlb_dns" {
  value = module.nlb.dns
}

output "rds_endpoint" {
  value = module.rds.endpoint
}

output "s3_bucket" {
  value = module.s3.nombre_bucket
}

output "ec2_ids" {
  value = module.ec2.ids
}

output "ec2_ips_publicas" {
  value = module.ec2.ips_publicas
}

output "vpc_id" {
  value = module.vpc.id_vpc
}

output "subredes_publicas" {
  value = module.vpc.subredes_publicas_ids
}

output "subredes_privadas" {
  value = module.vpc.subredes_privadas_ids
}
```

La entrada `key_name` debe coincidir con el nombre de un Key Pair existente en `us-east-1`. En `ssh_cidr` se escribe la IP pública de origen seguida de `/32`: el valor `203.0.113.10/32` es de ejemplo, no corresponde a tu conexión. La contraseña de RDS no va en `terraform.tfvars`.

**`.gitignore`**

```gitignore
.terraform/
*.tfstate
*.tfstate.*
*.tfplan
*.tfvars
*.tfvars.json
.env
*.pem
*.key
crash.log
crash.*.log
terraform-debug.log
*.log
```

## Ejecutar y comprobar

Todo se ejecuta desde la **carpeta raíz**, en la terminal **PowerShell de VS Code**, excepto los comandos de Linux indicados al entrar por SSH. Antes de iniciar Terraform, abre AWS Academy e inicia el laboratorio.

**Credenciales.** `aws configure` guarda credenciales en un perfil de AWS CLI, pero **no es obligatorio hacerlo**: Terraform AWS Provider también reconoce las siguientes variables de entorno. En AWS Academy son preferibles porque las credenciales y el token cambian al reiniciar el laboratorio. Se aplican a esta ventana de PowerShell; si se cierra, hay que volver a cargarlas. Si `aws configure` ya guardó credenciales anteriores, estas variables de entorno tienen prioridad durante la sesión.

```powershell
terraform -version
$env:AWS_ACCESS_KEY_ID="ACCESS_KEY_DEL_LAB"
$env:AWS_SECRET_ACCESS_KEY="SECRET_KEY_DEL_LAB"
$env:AWS_SESSION_TOKEN="SESSION_TOKEN_DEL_LAB"
$env:AWS_DEFAULT_REGION="us-east-1"
```

`$env:` indica una variable de entorno de PowerShell. No es una variable de Terraform ni se escribe en un `.tf`. `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY` y `AWS_SESSION_TOKEN` son las credenciales temporales. La región utilizada para crear los recursos también está definida en `provider.tf` mediante `var.region`.

**Contraseña RDS.** Terraform reconocerá automáticamente `TF_VAR_db_password` como el valor de la variable `db_password`. Estos dos comandos permiten introducirla sin mostrar lo que se escribe en pantalla:

```powershell
$pass = Read-Host "Contraseña de MySQL" -AsSecureString
$env:TF_VAR_db_password = [System.Net.NetworkCredential]::new("", $pass).Password
```

`$pass` es solo una variable temporal de **PowerShell** que guarda el dato introducido de forma protegida. La segunda línea lo convierte al texto que Terraform necesita y lo coloca en la variable de entorno. No es obligatorio llamarla `$pass`; tampoco tiene relación con la contraseña de la cuenta AWS. Usa una contraseña de 8 a 41 caracteres, evitando espacios, comillas dobles, `@` y `/` por las restricciones de RDS. Aunque `sensitive = true` la oculte en algunas salidas, puede quedar almacenada en el estado de Terraform.

**Preparación y despliegue en `dev`.** Reemplaza antes `key_name` y `ssh_cidr` en `terraform.tfvars`. Los comandos van en este orden:

```powershell
terraform init
terraform fmt -recursive
terraform validate
terraform plan -out=dev.tfplan
terraform apply dev.tfplan
terraform output
```

`init` descarga providers; `fmt` ordena el formato; `validate` revisa la configuración; `plan` muestra los cambios y `apply` aplica el plan guardado. Antes de ejecutar `apply`, revisa que realmente se creen la VPC, cuatro subredes, tres EC2, NLB, RDS y S3. Que `validate` termine correctamente no demuestra que AWS permita crear los servicios.

**NLB y página HTTP.** Después del despliegue, en AWS Console → EC2 → Target Groups revisa que estén registrados los tres destinos. En PowerShell:

```powershell
$dns = terraform output -raw nlb_dns
curl.exe "http://$dns/"
```

`$dns` es otra variable local de PowerShell: guarda el DNS que devuelve el output de Terraform. La respuesta debe contener el nombre de uno de los servidores; comprueba que el NLB llega al Apache de EC2, no que la biblioteca ya use RDS o S3.

**SSH y MySQL.** Consulta las IP de EC2 desde PowerShell y reemplaza los marcadores del comando SSH:

```powershell
terraform output ec2_ips_publicas
ssh -i "C:\ruta\clave.pem" ec2-user@IP_EC2
```

Una vez conectado a la EC2, estás en **Linux**, ya no en PowerShell. El output `rds_endpoint` muestra un DNS seguido de `:3306`; utiliza solo el DNS en `HOST_RDS`:

```bash
sudo dnf install -y mariadb105
mysql -h HOST_RDS -P 3306 -u admin -p -e "SELECT VERSION();"
```

El cliente solicitará la contraseña. Obtener una versión de MySQL demuestra conexión desde esa EC2. Para revisar que RDS no esté expuesta directamente, confirma en AWS Console `Publicly accessible: No` y que el SG de RDS solo permite 3306 desde el SG de EC2. Una conexión fallida desde Windows, aislada, no es prueba suficiente de seguridad.

**S3.** En AWS Console → S3 abre el bucket que muestra `terraform output -raw s3_bucket`, sube un TXT y descárgalo. Eso comprueba acceso autenticado de tu cuenta; no significa que el objeto sea público. Con AWS CLI instalada, puedes hacer la misma prueba en PowerShell:

```powershell
$bucket = terraform output -raw s3_bucket
"Prueba de Biblioteca Digital" | Set-Content .\prueba.txt
aws s3 cp .\prueba.txt "s3://$bucket/prueba.txt"
aws s3 cp "s3://$bucket/prueba.txt" .\prueba-descargada.txt
Get-Content .\prueba-descargada.txt
```

`$bucket` almacena el nombre del bucket obtenido de Terraform. `aws s3 cp` es un comando de **AWS CLI**; no funcionará si esa herramienta no está instalada. La consola de S3 es suficiente para esta prueba.

**Comparar `prod` sin desplegar.** El argumento `-var` tiene prioridad sobre el valor `ambiente = "dev"` de `terraform.tfvars` solo para ese comando:

```powershell
terraform plan -var="ambiente=prod"
```

Debe aparecer RDS Multi-AZ, discos de 60/60/120 GiB y versionado S3. Como el ambiente forma parte de los nombres, el plan también puede proponer sustituciones; **no ejecutar `apply` con este plan**.

**Limpieza del laboratorio.** Volver a `dev` (el valor del archivo sigue igual), ejecutar la destrucción y comprobar en AWS Console que no queden recursos:

```powershell
terraform destroy
Remove-Item Env:TF_VAR_db_password
```

`destroy` elimina recursos y puede borrar objetos S3 porque el bucket tiene `force_destroy = true`. No subir `terraform.tfstate`, `dev.tfplan`, claves ni credenciales a GitHub.

## Cambios de configuración

**Subredes.** Los mapas `subredes_publicas` y `subredes_privadas` están en `variables.tf` de la raíz. Actualmente contienen dos entradas cada uno. Para cambiar CIDR, zona o cantidad, se modifican esos mapas o se escriben **completos** en `terraform.tfvars`. Una entrada nueva se agrega junto a las existentes, con un CIDR que no se superponga. `modulos/vpc/main.tf` utiliza `for_each`, por lo que no hay que copiar bloques `aws_subnet`.

Las EC2 usan las claves `publica1` y `publica2` del mapa público. El NLB recibe sus IDs con `values(module.vpc.subredes_publicas_ids)` y necesita como máximo una subred por zona de disponibilidad. RDS recibe las privadas; para mantenerla, debe haber subredes en al menos dos zonas distintas. Si se quita una subred, primero hay que revisar qué servidores y módulos la utilizan.

**Cantidad y ubicación de EC2.** Las entradas están dentro de `servidores` en `terraform.tfvars`. Para dejar dos, se elimina una entrada completa. Para crear cuatro, se mantiene lo existente y se agrega, por ejemplo:

```hcl
biblioteca-d = {
  instance_type = "t3.medium"
  disk_size     = 30
  subnet        = "publica2"
}
```

Este bloque va **dentro de `servidores = { ... }`**, no como recurso nuevo. La clave `subnet` debe existir en `subredes_publicas`. `for_each` crea la EC2 y el NLB registra su ID sin modificar el módulo hijo. Los cambios en discos y tipos también se hacen en ese mapa; en `prod` el tamaño del disco se duplica por la condición del módulo EC2.

**Más NLB o buckets.** En el `main.tf` raíz, dentro de `module "nlb"` o `module "s3"`, se puede agregar `count`. Para dos NLB, se sustituye `nombre = local.nombre_base` por estas líneas:

```hcl
count  = 2
nombre = "${local.nombre_base}-${count.index + 1}"
```

Para tres buckets, las mismas líneas se colocan **dentro de `module "s3"`**, cambiando `count = 3`. Los demás argumentos de cada llamada se mantienen. Como ahora habrá varias instancias de módulo, en `outputs.tf` raíz cambian las referencias:

```hcl
# Dos NLB
value = module.nlb[*].dns

# Tres buckets
value = module.s3[*].nombre_bucket
```

Cada línea `value` reemplaza la del output correspondiente; no se agregan dos `value` en un mismo bloque. Los resultados pasan a ser listas, por lo que se consultan con `terraform output` sin `-raw`. Cambiar a `count` sobre módulos ya desplegados altera sus direcciones en el estado; revisar el plan.

**RDS.** El motor está en `modulos/rds/main.tf`. Para PostgreSQL cambian `engine`, `port` (5432), la regla de ingreso de `seguridad.tf` (5432) y el cliente de prueba (`psql` en lugar de `mysql`). La versión requerida debe estar disponible en AWS Academy y ser compatible con la clase de instancia elegida.

**Variables y Random.** El sufijo se ajusta en `main.tf` raíz, en `random_string.sufijo` (`length`). Los valores `nombre_base` y `tags_comunes` están en `locals`. Si se trabaja sin `default`, se retiran los valores predeterminados de `variables.tf` raíz y se pasan **todas** las entradas necesarias mediante `dev.tfvars`:

```powershell
terraform plan -var-file="dev.tfvars"
terraform apply -var-file="dev.tfvars"
```

La contraseña RDS sigue entrando por `TF_VAR_db_password`; no se escribe en `dev.tfvars` ni se sube a GitHub.

**Debugging.** Desde PowerShell, en la raíz, se puede guardar el registro de un `plan`:

```powershell
$env:TF_LOG="DEBUG"
$env:TF_LOG_PATH="$PWD\terraform-debug.log"
terraform plan
Remove-Item Env:TF_LOG
Remove-Item Env:TF_LOG_PATH
```

El archivo `.log` puede contener información sensible. No demuestra por sí solo que la infraestructura funciona. Antes de aplicar cualquiera de estos cambios, revisar `terraform validate` y `terraform plan`, especialmente si Terraform propone reemplazar o destruir recursos.
