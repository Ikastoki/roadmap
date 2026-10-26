# Mecanismos de Bootstrapping

Bootstrapping es el proceso de configuracion inicial que se ejecuta automaticamente cuando una instancia recien creada arranca por primera vez. El mecanismo mas extendido en la nube publica es **User Data** (AWS), **custom-data** (GCP) o **custom_data** (Azure), que permite inyectar un script o documento Cloud-init que se ejecuta en el primer boot del sistema operativo.

El flujo es el siguiente: al crear una instancia, se adjunta un bloque de datos al recurso. El agente de la nube (como `cloud-init` en AMIs basadas en Debian/Ubuntu) detecta esta informacion durante el arranque y ejecuta los comandos definidos. Esto ocurre una sola vez — en el primer boot — lo que lo convierte en un mecanismo ideal para configuraciones que no deben repetirse en cada reinicio.

## Cloud-init

`cloud-init` es el estandar de facto para bootstrapping en instancias Linux en la nube. Soporta multiples modulos: ejecucion de scripts, instalacion de paquetes, creacion de usuarios, escritura de archivos, ejecucion de comandos finales y mas. La sintaxis es YAML y se divide en bloques funcionales.

```yaml
#cloud-config
package_update: true
package_upgrade: true

packages:
  - nginx
  - curl
  - jq
  - htop

users:
  - name: deploy
    groups: sudo
    shell: /bin/bash
    sudo: ['ALL=(ALL) NOPASSWD:ALL']
    ssh_authorized_keys:
      - ssh-rsa AAAAB3... usuario@maquina

write_files:
  - path: /etc/nginx/conf.d/app.conf
    content: |
      server {
          listen 8080;
          server_name _;
          location / {
              proxy_pass http://127.0.0.1:3000;
              proxy_set_header Host $host;
              proxy_set_header X-Real-IP $remote_addr;
          }
      }

runcmd:
  - systemctl enable nginx
  - systemctl start nginx
  - echo "Bootstrap completado" > /var/log/bootstrap.log
```

> Cloud-init se ejecuta de forma idempotente en cuanto a paquetes (no reinstala lo ya instalado), pero los bloques `runcmd` se ejecutan en cada boot a menos que se marquen con `once: true` o se gestionen con `boothook`. Esto es importante para evitar efectos secundarios no deseados.

## User Data con Terraform

En Terraform, el provisionamiento con User Data se configura directamente en el recurso de instancia. Para escenarios simples, se puede usar un script Bash inline. Para escenarios mas estructurados, se prefiere renderizar un template de cloud-init.

```hcl
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"
  subnet_id     = aws_subnet.publica.id

  user_data = <<-EOF
              #!/bin/bash
              set -euxo pipefail

              apt-get update -y
              apt-get install -y nginx

              cat > /etc/nginx/sites-available/default <<'NGINX'
              server {
                  listen 80;
                  server_name _;
                  location / {
                      root /var/www/html;
                      index index.html;
                  }
              }
              NGINX

              systemctl restart nginx
              EOF

  tags = {
    Name = "servidor-web"
  }
}
```

Para escenarios mas complejos, Terraform permite renderizar un template de cloud-init con variables locales:

```hcl
locals {
  user_data = templatefile("${path.module}/templates/cloud-init.yaml", {
    db_host     = aws_instance.db.private_ip
    app_version = var.version_aplicacion
    entorno     = var.entorno
  })
}

resource "aws_instance" "app" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.small"
  user_data     = local.user_data

  tags = {
    Name = "servidor-app-${var.entorno}"
  }
}
```

```yaml
# templates/cloud-init.yaml
#cloud-config
package_update: true

packages:
  - python3
  - python3-pip
  - docker.io

write_files:
  - path: /opt/app/config.json
    content: |
      {
        "db_host": "${db_host}",
        "version": "${app_version}",
        "entorno": "${entorno}"
      }

runcmd:
  - docker pull mi-registro/app:${app_version}
  - docker run -d -p 3000:3000 --restart always mi-registro/app:${app_version}
```

## Limitaciones del bootstrapping

El bootstrapping tiene una ventaja clara: simplicidad. No requiere herramientas adicionales ni infraestructura de gestion de configuracion. Sin embargo, presenta limitaciones significativas en escenarios de produccion:

- **No hay estado**: el script se ejecuta una vez y no monitorea si la configuracion se mantiene correcta. Si alguien desinstala Nginx manualmente, no hay mecanismo de autocorreccion.
- **Fragilidad**: un error en el script puede dejar la instancia en un estado intermedio sin posibilidad de reintento automatico.
- **No soporta actualizaciones**: si se necesita cambiar la configuracion en una instancia ya existente, el User Data no se re-ejecuta (salvo que se fuerce un reboot y se use `boothook`).
- **Duplicacion de logica**: en multiples instancias, el mismo script se ejecuta N veces, lo que dificulta mantener versiones consistentes.

Para escenarios que requieren gestion continua del estado del SO, las herramientas de gestion de configuracion como Ansible, Chef o Puppet ofrecen un modelo mas robusto con ciclos de ejecucion periodicos, catalogos de estado y capacidad de autocorreccion.

