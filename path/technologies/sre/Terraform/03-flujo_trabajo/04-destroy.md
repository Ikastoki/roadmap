# Destruccion de infraestructura

`terraform destroy` es el comando que elimina todos los recursos gestionados por Terraform en una configuracion determinada. Aunque destruir infraestructura pueda parecer una operacion marginal, es una parte integral del ciclo de vida en entornos modernos donde los recursos se crean y eliminan con frecuencia: entornos de pruebas, laboratorios de formacion, preview environments asociados a pull requests o entornos temporales para realizar experimentos. La capacidad de destruir infraestructura de forma limpia y predecible es tan importante como la capacidad de crearla.

## terraform destroy

`terraform destroy` es funcionalmente equivalente a ejecutar `terraform apply` sobre una configuracion vacia. Terraform compara el estado actual con una configuracion que no contiene ningun recurso y genera un plan que destruye todo lo que existe. Por eso no es un comando magico ni especial: sigue el mismo mecanismo de plan y apply, con la misma logica de dependencias, grafo y orden de operaciones.

```bash
terraform destroy
```

Al igual que `terraform apply`, solicita confirmacion interactiva antes de proceder:

```
Plan: 0 to add, 0 to change, 2 to destroy.

Do you really want to destroy all resources?
  Terraform will destroy all your managed infrastructure, as shown above.
  There is no undo. Only 'yes' will be accepted to confirm.

  Enter a value: yes

aws_security_group.web: Destroying...
aws_instance.web: Destroying...
aws_instance.web: Destruction complete after 30s
aws_security_group.web: Destruction complete after 2s

Destroy complete! Resources: 2 destroyed.
```

Para entornos automatizados, existe la opcion `-auto-approve` que omite la confirmacion:

```bash
terraform destroy -auto-approve
```

> El orden de destruccion sigue el inverso del orden de creacion. Terraform destruye primero los recursos que dependen de otros, garantizando que no se intenta eliminar un recurso mientras otro lo esta utilizando. En el ejemplo anterior, la instancia EC2 se destruye antes que el grupo de seguridad porque la instancia depende del grupo de seguridad.

## Entornos temporales

Una de las aplicaciones mas potentes de `terraform destroy` es la gestion de entornos temporales (ephemeral environments). Estos entornos se crean bajo demanda para una tarea especifica, como validar un pull request o ejecutar un laboratorio de formacion, y se destruyen automaticamente cuando ya no son necesarios. Esta practica ahorra costes significativos en infraestructura cloud y evita la acumulacion de recursos olvidados.

```bash
# Crear entorno de pruebas
terraform workspace new pr-42
terraform apply -auto-approve

# Realizar las pruebas...

# Destruir entorno al finalizar
terraform destroy -auto-approve
terraform workspace select default
terraform workspace delete pr-42
```

La combinacion de workspaces con `terraform destroy` permite gestionar multiples entornos efimeros de forma limpia. Cada workspace tiene su propio estado, por lo que destruir un workspace no afecta a los demas.

## Destruccion selectiva

A veces no se desea destruir todos los recursos, sino solo un subconjunto. Terraform ofrece varias estrategias para lograr una destruccion selectiva sin necesidad de modificar la configuracion completa.

La opcion `-target` permite destruir recursos especificos:

```bash
terraform destroy -target=aws_instance.web
```

Eliminar un recurso del archivo de configuracion y ejecutar `terraform apply` tambien provoca su destruccion, pero solo del recurso eliminado, no del resto.

El meta-argumento `prevent_destroy` protege recursos contra eliminacion accidental:

```hcl
resource "aws_s3_bucket" "datos" {
  bucket = "datos-produccion"

  lifecycle {
    prevent_destroy = true
  }
}
```

Con `prevent_destroy` activo, cualquier intento de destruir el recurso (ya sea mediante `terraform destroy` o eliminando el bloque de configuracion) fallara con un error. Esta proteccion es especialmente util para bases de datos, buckets con informacion critica o cualquier recurso cuya eliminacion provocaria una perdida de datos irreversible.

> `prevent_destroy` no es una medida de seguridad absoluta. Un usuario con acceso al estado podria eliminar manualmente el recurso del estado y destruirlo fuera de Terraform. Es una barrera contra errores humanos, no contra acciones malintencionadas.

## Buenas practicas con destroy

Antes de ejecutar `terraform destroy` en produccion, conviene verificar que no se destruiran recursos criticos. Algunas practicas recomendadas incluyen:

Revisar siempre el plan generado por `destroy` antes de confirmar, prestando atencion a los recursos que se marcaran como destruidos. En entornos de equipo, el archivo de plan puede guardarse y compartirse para revision.

```bash
# Generar el plan de destruccion y revisarlo
terraform plan -destroy -out=destroy.tfplan
terraform show destroy.tfplan
# Solo cuando se haya verificado:
terraform apply destroy.tfplan
```

Mantener un inventario actualizado de los recursos gestionados por Terraform ayuda a entender el alcance de una destruccion. El comando `terraform state list` muestra todos los recursos bajo gestion y puede ejecutarse antes de cualquier operacion de destruccion para tener una vision clara de lo que se eliminara.

```bash
# Ver que recursos se destruiran antes de ejecutar destroy
terraform state list
```

> En entornos compartidos, el backend remoto con state locking previene que dos personas ejecuten `terraform destroy` simultaneamente sobre el mismo conjunto de recursos, lo que podria provocar condiciones de carrera y estados inconsistentes.

