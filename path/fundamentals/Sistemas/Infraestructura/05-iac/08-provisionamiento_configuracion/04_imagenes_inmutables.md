# Imagenes Inmutables: Eliminar la Configuracion en Runtime

El concepto de inmutabilidad en infraestructura propone que una vez que una imagen de sistema operativo esta construida, nunca se modifica en produccion. En lugar de levantar una instancia vacia y configurarla con herramientas externas, se genera una imagen pre-configurada con todo lo necesario para ejecutar la aplicacion: dependencias del sistema, paquetes, configuraciones, binarios compilados y incluso la propia aplicacion.

Este enfoque elimina por completo la necesidad de gestion de configuracion en tiempo de ejecucion. La instancia arranca lista para funcionar. No hay scripts de bootstrap, no hay agentes de Ansible ejecutandose periodicamente, no hay divergencia entre instancias. Si se necesita un cambio, se genera una nueva imagen y se despliega, no se modifica la instancia existente.

## Packer: construccion de imagenes

HashiCorp Packer es la herramienta estandar para construir imagenes de forma automatizada. Acepta templates en HCL que definen la imagen base, los provisioners que la configuran y los post-procesadores que la exportan a diferentes formatos (AMI en AWS, Image en GCP, Managed Image en Azure, Docker image, etc.).

```hcl
# webserver.pkr.hcl
packer {
  required_plugins {
    amazon = {
      version = ">= 1.2.0"
      source  = "github.com/hashicorp/amazon"
    }
  }
}

variable "version_app" {
  type    = string
  default = "latest"
}

source "amazon-ami" "webserver" {
  ami_name      = "webserver-{{user `version_app`}}-{{timestamp}}"
  instance_type = "t3.micro"
  source_ami_filter {
    filters = {
      name                = "ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*"
      root-device-type    = "ebs"
      virtualization-type = "hvm"
    }
    owners      = ["099720109477"]
    most_recent = true
  }
  ssh_username = "ubuntu"
  tags = {
    Version = "{{user `version_app`}}"
    Build   = "{{timestamp}}"
  }
}

build {
  sources = ["source.amazon-ami.webserver"]

  provisioner "shell" {
    inline = [
      "sudo apt-get update -y",
      "sudo apt-get install -y nginx python3 python3-pip",
      "sudo systemctl enable nginx"
    ]
  }

  provisioner "file" {
    source      = "configs/nginx.conf"
    destination = "/tmp/nginx.conf"
  }

  provisioner "shell" {
    inline = [
      "sudo mv /tmp/nginx.conf /etc/nginx/sites-available/default",
      "sudo systemctl restart nginx"
    ]
  }

  provisioner "ansible" {
    playbook_file = "playbooks/webserver.yml"
    extra_arguments = [
      "--extra-vars", "app_version=${var.version_app}"
    ]
  }

  post-processor "amazon-ami" {
    ami_name        = "webserver-final-${var.version_app}"
    ami_description = "Imagen webserver version ${var.version_app}"
  }
}
```

> Un patron comun es usar Packer con un provisioner de Ansible: se levanta una instancia temporal, se configura con los playbooks de Ansible que ya existen, y al finalizar se convierte en una AMI. Esto reutiliza la logica de configuracion existente sin duplicarla.

## Integracion con Terraform

Una vez generada la imagen, Terraform la consume como un recurso mas. La instancia se configura para usar la AMI especifica y la unica configuracion en runtime es el User Data minimo necesario (por ejemplo, inyectar variables de entorno o datos de conexión).

```hcl
data "aws_ami" "webserver" {
  most_recent = true
  owners      = ["self"]

  filter {
    name   = "name"
    values = ["webserver-${var.version_app}-*"]
  }

  filter {
    name   = "tag:Version"
    values = [var.version_app]
  }
}

resource "aws_launch_template" "web" {
  name          = "web-${var.version_app}"
  image_id      = data.aws_ami.webserver.id
  instance_type = "t3.medium"

  user_data = base64encode(templatefile("${path.module}/templates/user_data.yaml", {
    app_version = var.version_app
    entorno     = var.entorno
    db_endpoint = var.db_endpoint
  }))

  tag_specifications {
    resource_type = "instance"
    tags = {
      Name    = "web-${var.entorno}"
      Version = var.version_app
    }
  }
}

resource "aws_autoscaling_group" "web" {
  name                = "asg-web-${var.entorno}"
  min_size            = 2
  max_size            = 10
  desired_capacity    = 3
  vpc_zone_identifier = var.subnet_ids
  launch_template {
    id      = aws_launch_template.web.id
    version = "$Latest"
  }
}
```

El User Data en este caso es minimo porque la imagen ya contiene todo lo necesario:

```yaml
# templates/user_data.yaml
#cloud-config
write_files:
  - path: /opt/app/env.json
    content: |
      {
        "DB_ENDPOINT": "${db_endpoint}",
        "APP_VERSION": "${app_version}",
        "ENVIRONMENT": "${entorno}"
      }

runcmd:
  - systemctl start nginx
  - systemctl start app
```

## Pipeline completo

El ciclo de vida de una imagen inmutable en produccion sigue un flujo estructurado:

```
Codigo fuente + Configuracion
         |
         v
   +-----------+
   |  Packer   |  --> Construye imagen pre-configurada
   +-----------+
         |
         v
  Registro de imagenes (AMI / ECR / GCR)
         |
         v
   +-----------+
   | Terraform |  --> Despliega instancia usando la imagen
   +-----------+
         |
         v
  Infraestructura en produccion (sin configuracion adicional)
```

Este modelo elimina los problemas de configuracion en runtime pero introduce complejidad en el pipeline de construccion de imagenes. Cada cambio en la configuracion del SO requiere reconstruir la imagen y redesplegar las instancias, lo que demanda un proceso de CI/CD maduro y rapido. Para actualizaciones de seguridad criticas, sin embargo, este overhead se justifica: se genera una nueva imagen con los parches y se reemplazan las instancias existentes de forma controlada, en lugar de ejecutar `apt-get update` en cientos de nodos simultaneamente y esperar que todos queden en un estado consistente.

