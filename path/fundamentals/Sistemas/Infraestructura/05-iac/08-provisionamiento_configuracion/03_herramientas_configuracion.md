# Integracion con Herramientas de Gestion de Configuracion

Cuando el bootstrapping con scripts deja de ser suficiente — por complejidad de la configuracion, necesidad de autocorreccion o mantenimiento a largo plazo — las herramientas de gestion de configuracion entran como la capa complementaria. Estas herramientas no crean infraestructura; asumen que la infraestructura ya existe y se enfocan en mantener el estado deseado del sistema operativo y los servicios que ejecuta.

## Herramientas disponibles

Las cuatro herramientas principales en este espacio comparten un modelo conceptual comun pero difieren en su lenguaje, arquitectura y comunidad:

| Herramienta | Lenguaje | Arquitectura | Modelo de ejecucion |
|---|---|---|---|
| **Ansible** | YAML + Jinja2 | Agentless (SSH/WinRM) | Push periodico |
| **Puppet** | Puppet DSL (Ruby) | Agent en cada nodo | Pull del servidor catalogo |
| **Chef** | Ruby DSL | Agent (Chef Client) | Pull del servidor cookbook |
| **SaltStack** | YAML + Jinja2 | Agent (Minion) o SSH | Push o pull, extremadamente escalable |

**Ansible** se ha consolidado como la opcion mas popular para equipos que buscan simplicidad operativa. No requiere agentes instalados en los nodos目标 — se conecta via SSH y ejecuta modulos de forma remota. Su modelo basado en playbooks YAML lo hace accesible para equipos que ya usan YAML para otros aspectos de la infraestructura.

```yaml
# playbook de Ansible para configurar un servidor web
---
- name: Configurar servidor web
  hosts: webservers
  become: true
  vars:
    nginx_port: 8080
    app_user: deploy

  tasks:
    - name: Instalar dependencias del sistema
      apt:
        name:
          - nginx
          - python3
          - python3-pip
          - git
        state: present
        update_cache: yes

    - name: Crear usuario de despliegue
      user:
        name: "{{ app_user }}"
        shell: /bin/bash
        groups: www-data
        append: yes

    - name: Configurar Nginx
      template:
        src: templates/nginx.conf.j2
        dest: /etc/nginx/sites-available/default
      notify: Restart nginx

    - name: Activar sitio
      file:
        src: /etc/nginx/sites-available/default
        dest: /etc/nginx/sites-enabled/default
        state: link
      notify: Restart nginx

  handlers:
    - name: Restart nginx
      service:
        name: nginx
        state: restarted
```

> Ansible no mantiene estado entre ejecuciones, pero si soporta idempotencia: ejecutar el mismo playbook dos veces produce el mismo resultado sin errores. Esto lo hace seguro para ejecuciones periodicas via cron o CI/CD.

## Patron de integracion con IaC

El patron mas comun consiste en que IaC (Terraform) crea la infraestructura y luego invoca a la herramienta de configuracion. Existen multiples formas de hacer esta integracion:

### Local-exec provisioner

Terraform ejecuta un comando local despues de crear el recurso. Este patron es simple pero acopla el ciclo de vida de Terraform al de la herramienta de configuracion.

```hcl
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.medium"
  key_name      = aws_key_pair.deploy.key_name

  provisioner "local-exec" {
    command = <<-EOT
      ansible-playbook
        -i '${self.private_ip},'
        -u ubuntu
        --private-key ${var.ssh_key_path}
        -e 'app_version=${var.app_version}'
        playbooks/configurar_servidor.yml
    EOT

    environment = {
      ANSIBLE_HOST_KEY_CHECKING = "False"
    }
  }

  tags = {
    Name = "web-${var.entorno}"
  }
}
```

### Remote-exec provisioner

Terraform se conecta a la instancia remota via SSH y ejecuta comandos. En la practica, este patron se usa para instalar el agente de configuracion que luego toma el control.

```hcl
resource "aws_instance" "db" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "db.r5.large"

  provisioner "remote-exec" {
    inline = [
      "curl -fsSL https://packages.puppet.com/puppet7-release.deb | sudo dpkg -i -",
      "sudo apt-get update && sudo apt-get install -y puppet-agent",
      "sudo systemctl enable puppet && sudo systemctl start puppet"
    ]

    connection {
      type        = "ssh"
      user        = "ubuntu"
      private_key = file(var.ssh_key_path)
      host        = self.private_ip
    }
  }

  tags = {
    Name = "database-${var.entorno}"
  }
}
```

### Desacoplamiento via CI/CD (patron recomendado)

El patron mas robusto desacopla ambos procesos. Terraform gestiona la infraestructura y publica datos de salida (IPs, IDs). Una fase separada del pipeline de CI/CD consume esos datos y ejecuta la configuracion. Esto permite re-ejecutar la configuracion sin tocar la infraestructura.

```yaml
# pipeline GitLab CI/CD ejemplo
stages:
  - infra
  - config

terraform_apply:
  stage: infra
  script:
    - terraform init
    - terraform apply -auto-approve
    - terraform output -json > outputs.json
  artifacts:
    paths:
      - outputs.json

ansible_configure:
  stage: config
  script:
    - |
      IPs=$(jq -r '.webservers_ip.value[]' outputs.json)
      echo "$IPs" > inventario.txt
      ansible-playbook -i inventario.txt playbooks/web.yml
  needs:
    - terraform_apply
```

> El patron desacoplado tiene una ventaja adicional: si la configuracion falla, no afecta el estado de la infraestructura Terraform. Los recursos quedan provisionados y el pipeline puede reintentar solo la fase de configuracion sin riesgo de destruir nada.

