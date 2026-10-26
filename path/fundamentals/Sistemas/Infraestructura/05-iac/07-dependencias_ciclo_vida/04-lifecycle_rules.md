# Lifecycle rules

El ciclo de vida de un recurso en Terraform esta gobernado por tres operaciones fundamentales: creacion, actualizacion y destruccion. Sin embargo, el comportamiento predeterminado de estas operaciones no siempre se ajusta a los requisitos de infraestructura productiva. Un servidor de base de datos no puede simplemente eliminarse y recrearse si hay clientes conectados, un balanceador de carga en produccion no puede desaparecer momentaneamente durante una actualizacion de configuracion, y un bucket S3 con datos criticos no puede eliminarse accidentalmente por un operador distraido. El bloque `lifecycle` dentro de cada recurso de Terraform permite redefinir estas reglas de comportamiento, otorgando al operador control preciso sobre como Terraform gestiona cada fase del ciclo de vida de un recurso.

Las reglas de lifecycle no modifican el recurso en si ni alteran su configuracion. Lo que modifican es el algoritmo que Terraform aplica para sincronizar el estado deseado (la configuracion) con el estado real (la infraestructura existente). Cada regla responde a una necesidad distinta: evitar interrupciones de servicio, proteger contra eliminaciones accidentales o ignorar cambios que ocurren fuera del control de Terraform. Comprender cuando aplicar cada regla es esencial para diseñar despliegues seguros y robustos.

## Create before destroy

Cuando Terraform determina que un recurso debe ser reemplazado (por cambio en un atributo que fuerza recreacion), el comportamiento predeterminado es destruir el recurso existente y luego crear uno nuevo. Esto genera un periodo de indisponibilidad entre la destruccion y la creacion, lo cual es inaceptable para servicios que requieren disponibilidad continua. La regla `create_before_destroy` invierte este orden: Terraform crea primero el nuevo recurso y solo despues de que este este operativo, destruye el anterior.

```hcl
resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = "t3.micro"

  lifecycle {
    create_before_destroy = true
  }
}
```

Este patron es critico para balanceadores de carga, servidores de aplicacion detras de un pool, y cualquier recurso que sirva trafico en produccion. Durante la transicion, ambos recursos existen simultaneamente, lo cual significa que el balanceador de carga puede enrutar trafico al nuevo servidor mientras el antiguo sigue operando. Una vez que el nuevo recurso esta healthy, Terraform destruye el anterior.

> `create_before_destroy` requiere que el recurso admita la coexistencia temporal de dos instancias del mismo tipo. No todos los servicios lo permiten: por ejemplo, un recurso con un nombre unico como un dominio DNS o un bucket S3 con un nombre global no puede tener dos instancias simultaneas.

## Prevent destroy

La regla `prevent_destroy` establece una barrera que impide que Terraform destruya un recurso bajo ninguna circunstancia. Si un plan de ejecucion incluye la destruccion de un recurso protegido con esta regla, Terraform aborta la operacion completa y muestra un error. Esto protege contra eliminaciones accidentales de recursos criticos como bases de datos en produccion, buckets con datos empresariales o configuraciones de red que soportan servicios de alta disponibilidad.

```hcl
resource "aws_db_instance" "produccion" {
  identifier        = "db-produccion"
  engine            = "postgres"
  engine_version    = "15.4"
  instance_class    = "db.r6g.xlarge"
  allocated_storage = 100

  lifecycle {
    prevent_destroy = true
  }
}
```

Con esta configuracion, si alguien intenta ejecutar `terraform destroy` o si una modificacion en la configuracion requiere eliminar y recrear la instancia de base de datos, Terraform se negara a proceder. La unica forma de eliminar un recurso protegido con `prevent_destroy` es eliminar manualmente la regla del bloque `lifecycle` o eliminar el recurso del estado con `terraform state rm` despues de una revision deliberada.

```
Error: Resource aws_db_instance.produccion cannot be destroyed

Resource aws_db_instance.produccion has lifecycle.prevent_destroy set,
but the plan calls to destroy this object. To force a destroy, remove
the lifecycle.prevent_destroy configuration from the resource.
```

> `prevent_destroy` no evita que Terraform actualice el recurso. Solo impide su destruccion. Si un cambio de configuracion requiere recreacion, Terraform fallara porque no puede destruir el recurso existente para crear uno nuevo.

## Ignore changes

Los recursos en produccion sufren modificaciones manuales, actualizaciones automaticas de parches de seguridad, cambios en atributos gestionados por servicios cloud internamente, y otras alteraciones que ocurren fuera del control de Terraform. Cuando Terraform detecta una diferencia entre el estado registrado y el estado real del recurso, intenta corregirla en el proximo plan. La regla `ignore_changes` le indica a Terraform que ignore cambios especificos en ciertos atributos, evitando que las modificaciones externas generen planes de ejecucion innecesarios o conflictos con la configuracion declarada.

```hcl
resource "aws_instance" "servidor" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = "t3.micro"

  tags = {
    Name = "servidor-web"
  }

  lifecycle {
    ignore_changes = [
      tags["ultima_actualizacion"],
      ami,
    ]
  }
}
```

En este ejemplo, si el tag `ultima_actualizacion` es modificado externamente por un script de auditoria o si Amazon lanza una nueva version de la AMI y Terraform detecta un cambio en el atributo `ami`, el plan no incluira ninguna operacion sobre el recurso. Los cambios se registran en el estado pero no generan acciones correctivas.

Un caso de uso comun es cuando las instancias EC2 son actualizadas con SSM Patch Manager, que modifica la AMI de la instancia. Sin `ignore_changes`, Terraform intentaria revertir la instancia a la AMI original en cada plan, deshaciendo el parche aplicado por el sistema de gestion de parches.

```
# aws_instance.servidor will be updated in-place
  ~ ami = "ami-abc12345" -> "ami-def67890"
```

Sin `ignore_changes`, esta salida apareceria en cada ejecucion de `terraform plan` despues de que SSM Patch Manager actualizar la AMI. Con la regla activa, Terraform ignora el cambio y no genera ninguna operacion.

```hcl
resource "aws_autoscaling_group" "web" {
  name                = "web-asg"
  min_size            = 2
  max_size            = 10
  desired_capacity    = 3
  vpc_zone_identifier = [aws_subnet.publica.id]

  lifecycle {
    ignore_changes = [desired_capacity]
  }
}
```

Este patron es particularmente valioso cuando el Auto Scaling Group ajusta su capacidad de forma dinamica en respuesta a la carga. Terraform no intentara revertir la capacidad calculada por la politica de escalado, evitando conflictos entre la configuracion declarada y el comportamiento adaptativo del servicio.

## Interaccion entre reglas

Las reglas de lifecycle pueden combinarse dentro del mismo recurso, pero cada una opera de forma independiente. `create_before_destroy` y `prevent_destroy` son mutuamente excluyentes en la practica: si un recurso no puede destruirse, no tiene sentido crear uno nuevo antes de destruir el viejo porque la destruccion nunca ocurrira. `ignore_changes` puede coexistir con cualquiera de las otras dos reglas y es el que mas frecuentemente se combina con otros lifecycle directives.

```hcl
resource "aws_cloudfront_distribution" "cdn" {
  origin {
    domain_name = aws_s3_bucket.contenido.bucket_regional_domain_name
    origin_id   = "s3-contenido"
  }

  enabled = true

  default_cache_behavior {
    allowed_methods  = ["GET", "HEAD"]
    cached_methods   = ["GET", "HEAD"]
    target_origin_id = "s3-contenido"

    forwarded_values {
      query_string = false
      cookies {
        forward = "none"
      }
    }
  }

  restrictions {
    geo_restriction {
      restriction_type = "none"
    }
  }

  viewer_certificate {
    cloudfront_default_certificate = true
  }

  lifecycle {
    create_before_destroy = true
    ignore_changes = [
      default_cache_behavior["default_ttl"],
    ]
  }
}
```

En esta configuracion, CloudFront Distribution se reemplazara creando la nueva distribucion antes de eliminar la vieja (evitando interrupcion del servicio CDN), y al mismo tiempo ignorara cambios en el `default_ttl` que podrian ser modificados desde la consola de AWS o por politicas de cache automaticas. Estas dos reglas no entran en conflicto porque operan sobre dimensiones distintas del ciclo de vida: una controla el orden de reemplazo y la otra controla la deteccion de cambios.

