# Buenas Practicas en Pipelines

Un pipeline de Terraform no es simplemente una secuencia de comandos ejecutados en orden. Cada etapa tiene un proposito especifico dentro de la cadena de validacion y despliegue, y saltarse cualquiera de ellas erosiona la confianza en el proceso. Las siguientes practicas constituyen el estandar minimo aceptado en la industria para pipelines de infraestructura gestionada con Terraform.

## Validacion de Formato con terraform fmt -check

El primer filtro del pipeline debe comprobar que todo el codigo HCL sigue el formato canonico definido por el propio lenguaje. Terraform incluye un formateador oficial (`terraform fmt`) que normaliza la indentacion, la alineacion de asignaciones y el espaciado de los bloques. Ejecutar `terraform fmt -check` en modo de solo verificacion devuelve un codigo de salida distinto de cero si algun archivo no cumple el formato, lo que provoca el fallo del pipeline.

Esta regla puede parecer superficial, pero tiene implicaciones practicas importantes: un formato consistente elimina ruido en las revisiones de codigo, facilita la lectura de diffs y previene conflictos artificiales en merges cuando distintos desarrolladores usan configuraciones de editor diferentes. Lo ideal es integrar `terraform fmt` como hook de pre-commit ademas del pipeline, detectando problemas antes incluso de que el codigo llegue al repositorio remoto.

```bash
terraform fmt -check -recursive
```

> Si el pipeline falla por formato, el desarrollador ejecuta `terraform fmt` localmente y fuerza el push. Algunos equipos automatizan la correccion en el propio pipeline y hacen commit del resultado, aunque esta practica es controvertida porque mezcla cambios automaticos con el trabajo del desarrollador.

## Validacion de Sintaxis con terraform validate

`terraform validate` verifica que la configuracion es sintacticamente correcta y que las referencias entre recursos, modulos y variables son coherentes. Detecta errores como nombres de recursos inexistentes, tipos de datos incorrectos en variables o referencias a atributos que no existen en un recurso. A diferencia de `terraform fmt`, que solo comprueba el formato, `validate` requiere que el directorio este inicializado (`terraform init`), ya que necesita cargar los providers y los modulos para resolver las referencias.

El comando se ejecuta tras `terraform init` y antes de cualquier operacion que genere un plan. Esto evita consumir tiempo y recursos de computacion en planes que van a fallar por errores sintacticos.

```bash
terraform init
terraform validate
```

En pipelines, es recomendable separar `fmt -check` y `validate` en pasos distintos para que el fallo indique claramente si el problema es de formato o de sintaxis, facilitando la depuracion.

## Generacion del Plan con Salida a Archivo

El comando `terraform plan` es la operacion mas informativa del pipeline: muestra exactamente que recursos se crearan, modificaran o destruiran. Sin embargo, ejecutar `plan` sin el flag `-out` tiene una limitacion critica: el resultado solo se muestra en pantalla y no puede reutilizarse en un apply posterior. El pipeline debe almacenar el plan generado para que el mismo plan, con el mismo estado de referencia, se aplique mas tarde.

```bash
terraform plan -out=plan.tfplan
```

El archivo `plan.tfplan` contiene el resultado del plan en formato binario. Incluye no solo la lista de cambios, sino tambien las referencias al estado y los valores calculados. Almacenarlo como artefacto del pipeline garantiza que el apply posterior utiliza exactamente la misma informacion que se reviso, evitando la posibilidad de que el estado cambie entre el plan y el apply (por ejemplo, por un merge simultaneo de otro equipo).

```bash
# En el stage de plan
terraform plan -out=plan.tfplan

# En el stage de apply
terraform apply plan.tfplan
```

Los artefactos deben configurarse con un tiempo de expiracion adecuado (por ejemplo, 30 dias) y los planes para entornos efimeros como entornos de revision o feature branches pueden descartarse tras el merge.

## Revision Humana del Plan

Ningun pipeline deberia aplicar cambios en produccion sin una revision humana del plan. La automatizacion llega hasta la generacion del plan; la decision de ejecutarlo requiere criterio contextual que una maquina no puede evaluar: ¿es este cambio compatible con el trafico en vivo? ¿Coincide con el cambio esperado en la tarea? ¿Hay recursos que se van a destruir inesperadamente?

La revision se implementa de dos formas complementarias:

- **Comentario automatico en la PR**: herramientas como Atlantis, Terragrunt o scripts personalizados publican el resultado del plan como comentario en la pull request. El revisor analiza el plan en el mismo contexto que el codigo de la aplicacion.
- **Aprobacion manual en el pipeline**: el stage de apply se configura con una compuerta de aprobacion. En GitHub Actions mediante `environment`, en GitLab CI con `when: manual` y entornos protegidos, en Jenkins con el bloque `input`.

La combinacion de ambas (plan visible en la PR + aprobacion manual en el pipeline) es el estandar de maxima seguridad: cualquiera puede ver el plan, pero solo las personas autorizadas pueden ejecutarlo.

## Almacenamiento de Planes como Artefactos

Los planes de Terraform contienen informacion detallada sobre la infraestructura gestionada, incluyendo en ocasiones valores que pueden ser sensibles (como ARNs o IDs de recursos). Almacenarlos como artefactos del pipeline presenta varias ventajas:

- **Trazabilidad**: cada plan queda asociado a una ejecucion concreta del pipeline, con su commit, autor y momento de generacion.
- **Reutilizacion**: el mismo plan puede aplicarse de forma consistente sin necesidad de recalcularlo.
- **Auditoria**: en caso de incidente, el plan aplicado puede consultarse para determinar exactamente que cambios se ejecutaron.

Cada plataforma de CI/CD tiene su mecanismo de artefactos:

```yaml
# GitHub Actions
- uses: actions/upload-artifact@v4
  with:
    name: terraform-plan
    path: ./infra/plan.tfplan
```

```yaml
# GitLab CI
artifacts:
  paths:
    - ${TF_ROOT}/${TF_PLAN_NAME}
  expire_in: 30 days
```

```groovy
// Jenkins
stash name: 'plan', includes: "**/plan.tfplan"
```

> Los planes binarios no son legibles por humanos directamente, pero pueden inspeccionarse con `terraform show plan.tfplan` si es necesario consultar su contenido tras la ejecucion.

## Orden de Etapas Recomendado

La secuencia completa de etapas en un pipeline de Terraform, ordenadas de menor a mayor coste y riesgo, es:

1. **`terraform fmt -check`** — validacion de formato (segundos, sin dependencias)
2. **`terraform init`** — inicializacion y descarga de providers
3. **`terraform validate`** — validacion sintactica y de referencias
4. **`terraform plan -out=plan.tfplan`** — generacion del plan y almacenamiento como artefacto
5. **Revision humana** — comentario en PR o aprobacion manual
6. **`terraform apply plan.tfplan`** — ejecucion del plan previamente aprobado

Cualquier desviacion de este orden introduce riesgo. Ejecutar `apply` sin `plan` previo elimina la posibilidad de revision. Almacenar el plan sin expiracion acumula basura. Validar sin `fmt -check` permite que el repositorio acumule incoherencias de formato que dificultan las revisiones.

