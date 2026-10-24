# Ciclo de vida de recursos

Terraform permite controlar el comportamiento de cada recurso durante los cambios mediante el bloque `lifecycle`, anidado dentro de la definicion del recurso. Las tres directivas principales —`create_before_destroy`, `prevent_destroy` e `ignore_changes`— cubren escenarios que van desde despliegues sin corte de servicio hasta proteccion contra eliminacion accidental. No son directivas de configuracion de infraestructura, sino de gestion de cambios: modifican como Terraform transiciona entre el estado actual y el estado deseado.

## create_before_destroy

Por defecto, cuando Terraform detecta que un recurso debe ser reemplazado (porque un cambio en un atributo obliga a recrearlo), primero destruye el recurso existente y luego crea el nuevo. Esta secuencia es segura desde la perspectiva del estado, pero puede provocar cortes de servicio en recursos como balanceadores, instancias detras de un auto scaling group o bases de datos.

`create_before_destroy` invierte el orden: primero se crea el nuevo recurso, luego se actualizan las referencias (si es necesario), y finalmente se destruye el recurso antiguo. Esto permite migraciones sin ventana de indisponibilidad siempre que el proveedor cloud soporte la coexistencia temporal de ambos recursos.

```hcl
resource "aws_instance" "app" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"

  lifecycle {
    create_before_destroy = true
  }
}
```

Cuando se aplica con `create_before_destroy`, Terraform genera un nombre unico temporal para el nuevo recurso, lo crea, lo registra en el estado y luego elimina el antiguo. Para recursos cuyo nombre debe ser unico (como un bucket S3), Terraform anade un sufijo aleatorio durante el periodo de transicion para evitar colisiones.

```hcl
resource "aws_lb" "web" {
  name = "web-lb"

  lifecycle {
    create_before_destroy = true
  }
}
```

En este caso, Terraform creara temporalmente `web-lb-new` junto a `web-lb`, reconfigurara los target groups si es necesario, y luego eliminara `web-lb`. El resultado final es que el nombre vuelve a ser `web-lb` tras la migracion.

> `create_before_destroy` tiene implicaciones en los recursos que dependen del recurso reemplazado. Si un security group depende de una VPC y la VPC usa `create_before_destroy`, Terraform debe propagar ese comportamiento a los recursos dependientes, creando tambien el nuevo security group asociado a la nueva VPC antes de destruir los antiguos. En algunos casos esto puede generar conflictos de nombres o cuotas.

## prevent_destroy

La directiva `prevent_destroy` es una salvaguarda contra la eliminacion accidental de recursos criticos. Cuando esta activa, cualquier ejecucion de `terraform destroy` o `terraform apply` que intente eliminar el recurso fallara con un error en lugar de continuar.

```hcl
resource "aws_s3_bucket" "logs" {
  bucket = "logs-produccion"

  lifecycle {
    prevent_destroy = true
  }
}
```

Cualquier intento de destruir el bucket producira un error similar a:

```
Error: Instance cannot be destroyed
  on main.tf line 10:
  10: resource "aws_s3_bucket" "logs" {
  Resource aws_s3_bucket.logs has lifecycle.prevent_destroy set.
```

Para eliminar el recurso, es necesario eliminar manualmente el bloque `prevent_destroy` del codigo y volver a ejecutar `terraform apply`. Esto fuerza una decision consciente del operador, evitando que comandos automatizados o errores humanos destruyan recursos que no deberian eliminarse.

> `prevent_destroy` tambien bloquea la recreacion (replacement). Si un recurso con `prevent_destroy` necesita ser reemplazado por un cambio en sus atributos, Terraform no podra aplicar el plan. En esos casos, debe eliminarse temporalmente la proteccion, aplicar el plan (que recreara el recurso) y volver a anadir la proteccion.

La directiva es especialmente util en recursos como buckets de logs, bases de datos de produccion, balanceadores que sirven trafico real o cualquier recurso cuya eliminacion causaria una perdida de datos o una interrupcion significativa. No debe abusarse: aplicarla a cada recurso innecesariamente dificulta la gestion del ciclo de vida.

## ignore_changes

`ignore_changes` permite especificar atributos del recurso que Terraform no debe considerar al calcular diferencias entre el estado y la configuracion. Es util cuando un atributo es modificado externamente a Terraform y no se desea que vuelva a su valor definido en el codigo.

```hcl
resource "aws_instance" "app" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"

  tags = {
    Name = "app-server"
    Environment = "production"
  }

  lifecycle {
    ignore_changes = [
      tags,
      ami,
    ]
  }
}
```

Con esta configuracion, si alguien anade o modifica etiquetas manualmente desde la consola AWS, Terraform no las revertira en el siguiente `apply`. Del mismo modo, si la AMI se actualiza mediante un proceso externo (por ejemplo, un pipeline de Packer que la reemplaza), Terraform no forzara el cambio de vuelta a la AMI original.

El caso de uso mas comun son los auto scaling groups, donde el numero de instancias (`desired_capacity`) puede ser modificado por el escalado automatico, y no debe ser sobrescrito por Terraform en cada ejecucion:

```hcl
resource "aws_autoscaling_group" "app" {
  desired_capacity = 3
  min_size         = 1
  max_size         = 10

  lifecycle {
    ignore_changes = [
      desired_capacity,
    ]
  }
}
```

> `ignore_changes` no evita que Terraform cree el recurso con los valores especificados inicialmente. Solo afecta a ejecuciones posteriores: si el atributo cambia externamente, Terraform lo ignora. Si el atributo se elimina del `ignore_changes` en una ejecucion futura, Terraform detectara la diferencia y podria modificarlo.

Es posible usar `ignore_changes = all` para que Terraform ignore cualquier cambio en todos los atributos del recurso, pero esto es una mala practica porque oculta desviaciones que podrian indicar problemas de configuracion o seguridad. Es mejor especificar explicitamente los atributos que se espera que cambien externamente.

## Interaccion entre directivas

Las tres directivas pueden combinarse dentro del mismo bloque `lifecycle`, aunque con ciertas restricciones. `create_before_destroy` y `prevent_destroy` son logicamente incompatibles: no se puede crear un reemplazo antes de destruir si la destruccion esta prohibida. Terraform producira un error si se intenta usar ambas simultaneamente.

```hcl
lifecycle {
  create_before_destroy = true
  prevent_destroy       = false  # necesario para que funcione
}
```

`ignore_changes` funciona de forma independiente: no interactua directamente con las otras dos directivas, pero puede influir en si un recurso se considera modificado y, por tanto, si necesita ser reemplazado o actualizado. Por ejemplo, ignorar cambios en `ami` evita que un cambio de AMI dispare un reemplazo, lo que a su vez hace que `create_before_destroy` no se active para ese cambio en concreto.

El bloque `lifecycle` es una herramienta de gestion de cambios, no de configuracion de recursos. Su uso correcto requiere entender el comportamiento del recurso subyacente y las implicaciones de cada directiva en el contexto del proveedor cloud y la arquitectura general.

