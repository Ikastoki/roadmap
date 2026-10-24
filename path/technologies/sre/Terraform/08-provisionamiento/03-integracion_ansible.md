# Integracion con Ansible y Chef

Terraform y las herramientas de gestion de configuracion como Ansible, Chef o Puppet ocupan espacios complementarios dentro del ciclo de vida de la infraestructura. Terraform se encarga del **aprovisionamiento** (provisioning): crear, modificar y destruir recursos de infraestructura como redes, instancias, balanceadores y bases de datos. Ansible, Chef y Puppet se encargan de la **configuracion** (configuration management): instalar software, desplegar aplicaciones, gestionar usuarios y mantener el estado deseado del software dentro de las instancias ya creadas.

El uso combinado sigue el principio de separacion de responsabilidades: Terraform declara la infraestructura como recursos de API, mientras que las herramientas de configuracion trabajan dentro de esa infraestructura. Mezclar ambas responsabilidades en una unica herramienta produce configuraciones acopladas y dificiles de mantener.

## Patron de integracion basico

La integracion tipica sigue tres fases:

1. Terraform crea la infraestructura (VPC, subredes, instancias, grupos de seguridad).
2. Terraform genera un inventario o archivo de configuracion con las direcciones IP, nombres de host y metadatos de las instancias.
3. Terraform invoca Ansible (o la herramienta correspondiente) mediante un `local-exec` provisioner, pasandole el inventario generado.

```hcl
resource "aws_instance" "web" {
  count         = 3
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"

  tags = {
    Name = "web-${count.index}"
    Role = "web"
  }
}

resource "local_file" "inventario" {
  filename = "${path.module}/ansible/inventario.ini"
  content  = <<-EOF
[web]
%{ for i in aws_instance.web ~}
${i.public_ip} ansible_user=ubuntu host_name=${i.tags.Name}
%{ endfor ~}
EOF
}

resource "null_resource" "ansible_provisioner" {
  depends_on = [aws_instance.web, local_file.inventario]

  provisioner "local-exec" {
    command = "ansible-playbook -i ansible/inventario.ini ansible/playbook.yml"
  }
}
```

El recurso `null_resource` actua como un contenedor para el provisioner que ejecuta Ansible. Como `null_resource` no gestiona ningun recurso real, se usa exclusivamente como disparador de acciones. La clave `triggers` permite forzar su re-ejecucion cuando cambian las instancias:

```hcl
resource "null_resource" "ansible_provisioner" {
  triggers = {
    instance_ids = join(",", aws_instance.web[*].id)
  }

  provisioner "local-exec" {
    command = "ansible-playbook -i ansible/inventario.ini ansible/playbook.yml"
  }
}
```

Cuando el contenido de `triggers` cambia (por ejemplo, tras reemplazar una instancia), Terraform destruye el `null_resource` y lo vuelve a crear, ejecutando el provisioner de nuevo.

## Manejo de claves SSH

Ansible necesita acceso SSH a las instancias. La forma mas segura de gestionar las claves es delegar la generacion a Terraform y pasarsela a Ansible mediante variables de entorno o archivos temporales:

```hcl
resource "tls_private_key" "deployer" {
  algorithm = "RSA"
  rsa_bits  = 4096
}

resource "aws_key_pair" "deployer" {
  key_name   = "deployer-key"
  public_key = tls_private_key.deployer.public_key_openssh
}

resource "local_sensitive_file" "private_key" {
  filename        = "${path.module}/ansible/ssh_key.pem"
  content         = tls_private_key.deployer.private_key_pem
  file_permission = "0600"
}

resource "null_resource" "ansible_provisioner" {
  depends_on = [aws_instance.web, local_sensitive_file.private_key]

  provisioner "local-exec" {
    command = "ansible-playbook -i ansible/inventario.ini ansible/playbook.yml --private-key ansible/ssh_key.pem"
  }
}
```

> El archivo de clave privada no debe versionarse en Git. Incluir `ansible/ssh_key.pem` en `.gitignore` y usar un backend remoto para el estado de Terraform que pueda contener datos sensibles.

## Integracion con Chef

Con Chef el patron es similar, pero en lugar de un playbook se invoca `chef-solo` o `knife`:

```hcl
resource "null_resource" "chef_provisioner" {
  depends_on = [aws_instance.web]

  provisioner "local-exec" {
    command = <<-EOC
      knife bootstrap ${aws_instance.web[0].public_ip} \
        --ssh-user ubuntu \
        --sudo \
        --run-list 'role[web_server]'
    EOC
  }
}
```

Chef tambien puede ejecutarse en modo local (chef-solo) subiendo previamente el repositorio de recetas via `file` provisioner y ejecutando `chef-solo` via `remote-exec`. Sin embargo, el patron de invocacion desde local-exec es preferible porque mantiene la logica de orquestacion en la maquina que ejecuta Terraform.

## Consideraciones sobre idempotencia y estado

El principal desafio al integrar Terraform con herramientas de configuracion es la gestion del estado entre ambas. Terraform no tiene conocimiento de lo que Ansible hace dentro de las instancias. Si se ejecuta `terraform apply` dos veces seguidas sin que nada cambie en la infraestructura, el `null_resource` no se vuelve a ejecutar (a menos que sus `triggers` hayan cambiado), y Ansible no se invoca de nuevo. Esto es correcto si la configuracion ya esta aplicada, pero puede ser problematico si:

- El playbook de Ansible cambia pero la infraestructura no (los triggers de `null_resource` no se modifican).
- Una instancia se corrompe y necesita reconfiguracion sin ser reemplazada.

Para el primer caso, se puede anadir una variable de version del playbook como trigger:

```hcl
variable "playbook_version" {
  description = "Version del playbook para forzar re-ejecucion"
  type        = string
}

resource "null_resource" "ansible_provisioner" {
  triggers = {
    instance_ids    = join(",", aws_instance.web[*].id)
    playbook_sha256 = filesha256("${path.module}/ansible/playbook.yml")
  }

  provisioner "local-exec" {
    command = "ansible-playbook -i ansible/inventario.ini ansible/playbook.yml"
  }
}
```

Con `filesha256` aplicado al playbook, cualquier cambio en el archivo modifica los triggers y fuerza la re-ejecucion del `null_resource`.

## Alternativa: Terraform provider para Ansible

Existe un provider de Terraform para Ansible (mantenido por la comunidad) que permite modelar la ejecucion de playbooks como recursos declarativos:

```hcl
terraform {
  required_providers {
    ansible = {
      source = "ansible/ansible"
    }
  }
}

resource "ansible_playbook" "web" {
  playbook   = "ansible/playbook.yml"
  inventory  = "ansible/inventario.ini"
  extra_vars = {
    environment = var.environment
  }
}
```

Este enfoque integra Ansible en el grafo de dependencias de Terraform, permitiendo que Terraform gestione la ejecucion de forma declarativa. Sin embargo, el provider no es oficial de HashiCorp y su adopcion en produccion debe evaluarse segun el nivel de soporte requerido.

## Buenas practicas

1. **Separacion de responsabilidades**: Terraform solo crea infraestructura. Ansible solo configura el software dentro de ella. No mezclar roles.
2. **Estado compartido**: si es posible, usar el backend remoto de Terraform para que los playbooks de Ansible puedan consultar el estado (`terraform output`) y actuar en consecuencia.
3. **Versionado**: mantener playbooks de Ansible y configuracion de Terraform en el mismo repositorio o en repositorios versionados sincronizados.
4. **Testing**: probar los playbooks de forma independiente antes de integrarlos en el pipeline de Terraform.
5. **Tiempo de espera**: las ejecuciones de Ansible pueden ser largas. Configurar `timeout` adecuado en el provisioner `local-exec` para evitar falsos fallos.

```hcl
provisioner "local-exec" {
  command = "ansible-playbook -i inventario.ini playbook.yml"
  timeout = "10m"
}
```

