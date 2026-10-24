# Provisioners en Terraform

Terraform esta disenado como una herramienta declarativa: se describe el estado deseado y el motor interno calcula el plan de acciones necesario para alcanzarlo. Sin embargo, existen escenarios donde es necesario ejecutar acciones imperativas durante el ciclo de vida de los recursos, ya sea para copiar archivos, ejecutar comandos en instancias recien creadas o disparar procesos externos. Para estos casos, Terraform ofrece los **provisioners**, bloques que permiten ejecutar logica procedural dentro de un flujo declarativo.

Los provisioners se definen dentro del bloque de un recurso y se ejecutan en momentos concretos: tras la creacion (`creation-time`), antes de la destruccion (`destroy-time`) o tras el resto de provisioners (`when = destroy`). Todos ellos requieren que el recurso exista y, en el caso de los remotos, que Terraform pueda establecer conexion con la maquina destino.

## Bloque connection

Los provisioners `file` y `remote-exec` necesitan un bloque `connection` que defina como acceder a la maquina remota. Los transportes soportados son SSH (por defecto) y WinRM:

```hcl
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"

  connection {
    type        = "ssh"
    host        = self.public_ip
    user        = "ubuntu"
    private_key = file("~/.ssh/id_rsa")
    timeout     = "5m"
  }

  provisioner "file" {
    source      = "app.conf"
    destination = "/tmp/app.conf"
  }

  provisioner "remote-exec" {
    inline = [
      "sudo mv /tmp/app.conf /etc/myapp/app.conf",
      "sudo systemctl restart myapp"
    ]
  }
}
```

> La direccion IP puede no estar disponible inmediatamente tras la creacion de la instancia. Es recomendable usar `self.public_ip` combinado con un `timeout` suficiente para que el servicio DHCP asigne la IP antes de que Terraform intente la conexion.

## Provisioner file

Copia archivos o directorios desde la maquina local hacia la remota:

```hcl
provisioner "file" {
  source      = "${path.module}/configs/"
  destination = "/opt/myapp/config"
}
```

- `source` puede ser un archivo o un directorio (en cuyo caso se copia recursivamente).
- `destination` es siempre la ruta absoluta en el destino.
- El directorio destino debe existir en la maquina remota; si no existe, el provisioner falla.
- No soporta templates ni transformaciones; si se necesita interpolacion hay que usar `templatefile` previamente y pasar el resultado como contenido.

Para contenido generado dinamicamente se puede usar `content` en lugar de `source`:

```hcl
provisioner "file" {
  content     = templatefile("app.conf.tpl", { port = var.app_port })
  destination = "/opt/myapp/app.conf"
}
```

## Provisioner remote-exec

Ejecuta comandos en la maquina remota via SSH o WinRM:

```hcl
provisioner "remote-exec" {
  inline = [
    "sudo apt-get update -y",
    "sudo apt-get install -y nginx",
    "sudo systemctl start nginx"
  ]
}
```

Cuando los comandos son muchos o complejos, es mejor externalizarlos a un script:

```hcl
provisioner "remote-exec" {
  script = "${path.module}/scripts/install.sh"
}
```

Tambien se pueden enviar multiples scripts con `scripts` (notar la `s`):

```hcl
provisioner "remote-exec" {
  scripts = [
    "${path.module}/scripts/install_deps.sh",
    "${path.module}/scripts/configure_app.sh",
    "${path.module}/scripts/start_service.sh"
  ]
}
```

Los scripts se copian a la maquina remota y se ejecutan secuencialmente. Si un script falla (codigo de salida distinto de cero), el provisioner falla y Terraform marca el recurso como erroneo.

## Provisioner local-exec

Ejecuta comandos en la maquina donde se ejecuta Terraform, no en la remota:

```hcl
resource "aws_instance" "web" {
  # ...

  provisioner "local-exec" {
    command = "echo ${self.public_ip} >> ips.txt"
  }
}
```

Es util para:

- Registrar direcciones IP en un inventario
- Invocar herramientas externas (Ansible, curl, scripts personalizados)
- Notificar a servicios externos via API
- Actualizar registros DNS gestionados fuera de Terraform

El parametro `interpreter` permite especificar el shell o binario que ejecutara el comando:

```hcl
provisioner "local-exec" {
  command     = "Add-DnsServerResourceRecordA -Name web -Zone example.com -IPv4Address ${self.private_ip}"
  interpreter = ["powershell", "-Command"]
}
```

## Provisioners en destroy-time

Por defecto los provisioners se ejecutan solo durante la creacion. Con `when = destroy` se ejecutan antes de que el recurso sea destruido:

```hcl
resource "aws_instance" "web" {
  # ...

  provisioner "local-exec" {
    when    = destroy
    command = "curl -X POST -d 'host=${self.public_ip}' https://api.monitor.com/deregister"
  }
}
```

> Los destroy-provisioners tienen una limitacion importante: si el recurso ya no existe (por ejemplo, si se elimino manualmente fuera de Terraform), el provisioner falla y el `terraform destroy` se bloquea. En estos casos se puede usar `terraform destroy -target` o importar el recurso primero.

## Advertencias y alternativas

La documentacion oficial de HashiCorp es explicita: los provisioners son un **ultimo recurso** (`last resort`). Las razones son multiples:

- **Idempotencia**: un provisioner no es inherentemente idempotente. Ejecutar `terraform apply` dos veces puede comportarse de forma distinta si los comandos no contemplan la re-ejecucion.
- **Fallos parciales**: si el provisioner falla tras modificar parcialmente el sistema, el estado de Terraform registra el recurso como degradado, pero no hay mecanismo automatico de limpieza.
- **Dependencia temporal**: los provisioners introducen dependencias en tiempo de ejecucion que no forman parte del grafo de recursos. Terraform no puede paralelizar ni optimizar su ejecucion.
- **Portabilidad**: el uso de provisioners ata la configuracion a detalles especificos del SO (rutas, comandos, sistema de paquetes), reduciendo la reutilizacion entre entornos.

Las alternativas recomendadas son, en orden de preferencia:

1. **User data / cloud-init** para tareas de bootstrap en instancias cloud.
2. **Imagenes personalizadas (AMI/Packer)** para entornos que requieren configuracion exacta y repetible.
3. **Herramientas de gestion de configuracion** (Ansible, Chef, Puppet) para mantenimiento continuo.
4. **Provisioners** solo cuando no exista alternativa viable, y siempre con scripts idempotentes y manejo explicito de errores.

Si a pesar de todo se opta por usar provisioners, se recomienda:

- Usar scripts externos en lugar de comandos inline para facilitar el testing.
- Implementar comprobaciones de idempotencia al inicio del script.
- Limitar su uso a tareas puntuales y bien acotadas.
- Documentar explicitamente por que no se pudo usar una alternativa.

