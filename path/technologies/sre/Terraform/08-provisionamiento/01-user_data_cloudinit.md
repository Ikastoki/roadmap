# User data y Cloud-init

El momento inmediatamente posterior al arranque de una instancia es critico: el sistema operativo esta recien instalado, sin configuracion especifica de la aplicacion ni las dependencias necesarias para cumplir su funcion. Tradicionalmente, los operadores accedian via SSH para ejecutar manualmente los comandos de configuracion inicial, un proceso que no escala y que introduce desviaciones entre entornos. Los mecanismos de user data y cloud-init resuelven este problema al permitir que la propia instancia ejecute scripts de bootstrap en su primer arranque, de forma autonomica y sin intervencion externa.

User data es un campo presente en la mayoria de proveedores cloud (AWS, Azure, GCP) que permite inyectar contenido textual en una instancia en el momento de su creacion. Este contenido puede ser un script de shell, un archivo de configuracion o una directiva cloud-init. La instancia lo recibe a traves del servicio de metadatos interno (por ejemplo, `http://169.254.169.254/latest/user-data` en AWS) y lo ejecuta durante la secuencia de arranque, antes de que el sistema este completamente operativo.

## Cloud-init

Cloud-init es el estandar de facto para la configuracion inicial de instancias en entornos cloud. Es un paquete disponible en la mayoria de distribuciones Linux modernas (Ubuntu, CentOS, Amazon Linux, Debian) que se ejecuta durante el primer arranque y procesa directivas en formato YAML para realizar tareas como:

- Actualizar paquetes del sistema
- Instalar dependencias adicionales
- Crear usuarios y grupos
- Escribir archivos de configuracion
- Ejecutar comandos arbitrarios
- Configurar la red
- Registrar la instancia en servicios externos

Un archivo cloud-init basico utiliza el formato `#cloud-config` como primera linea:

```yaml
#cloud-config
package_update: true
package_upgrade: true

packages:
  - nginx
  - git
  - python3-pip

write_files:
  - path: /etc/nginx/sites-available/default
    content: |
      server {
          listen 80;
          root /var/www/html;
          index index.html;
      }

runcmd:
  - systemctl enable nginx
  - systemctl start nginx
  - echo "Bootstrap completado" > /var/log/bootstrap.log
```

La instruccion `runcmd` ejecuta los comandos en el orden especificado, pero no garantiza que los servicios esten completamente iniciados al finalizar. Para aplicaciones que requieren un orden estricto, es preferible usar `bootcmd` (se ejecuta antes de que cloud-init monte discos o configure red) o scripts separados con control de flujo.

## Uso con Terraform

En Terraform, el user data se especifica mediante el argumento `user_data` de los recursos de computo:

```hcl
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"

  user_data = <<-EOF
    #!/bin/bash
    apt-get update -y
    apt-get install -y nginx
    systemctl start nginx
    echo "Servidor web listo" > /var/www/html/index.html
  EOF

  tags = {
    Name = "web-server"
  }
}
```

Para cloud-init en formato YAML, se puede usar la funcion `templatefile` para separar la configuracion en un archivo independiente:

```hcl
resource "aws_instance" "app" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"

  user_data = templatefile("${path.module}/cloud-init.yml", {
    app_version = var.app_version
    environment = var.environment
  })
}
```

Y el archivo `cloud-init.yml` correspondiente:

```yaml
#cloud-config
package_update: true
packages:
  - docker.io
  - docker-compose-v2

runcmd:
  - systemctl start docker
  - docker pull myapp:${app_version}
  - docker run -d -p 80:3000 myapp:${app_version}
```

> El user_data solo se procesa en el primer arranque de la instancia. Si una instancia se detiene y se inicia de nuevo, el user_data no se re-ejecuta. Para forzar la re-ejecucion es necesario reemplazar la instancia o utilizar herramientas de gestion de configuracion posteriores.

## Limitaciones

El approach de user data y cloud-init presenta restricciones importantes:

- **No gestiona estado**: cada ejecucion es independiente y no mantiene informacion sobre ejecuciones anteriores. Si un paso falla, no hay mecanismo nativo de reintento inteligente.
- **Dificil de depurar**: los logs se almacenan en `/var/log/cloud-init-output.log` y `/var/log/cloud-init.log`, pero el acceso a estos requiere conexion a la instancia, que precisamente puede ser lo que fallo.
- **Idempotencia limitada**: un script de shell en user_data no es inherentemente idempotente. Si se ejecuta dos veces (por ejemplo, si se crea una AMI personalizada y luego se lanza una instancia), puede producir resultados inesperados.
- **Sin visibilidad**: Terraform no tiene forma de saber si el user_data se ejecuto correctamente. El recurso se marca como creado en cuanto la API cloud confirma la creacion de la instancia, independientemente del exito del script interno.

Por estas razones, user_data y cloud-init son adecuados para tareas de bootstrap inicial (instalar paquetes, descargar configuracion, registrar la instancia en un servicio de orquestacion), pero no deben ser el unico mecanismo de gestion de configuracion para entornos en produccion. Para eso existen herramientas como Ansible, Chef o Puppet, que se integran con Terraform en un paso posterior.

