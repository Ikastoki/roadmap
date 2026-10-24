# Validacion y testing

La infraestructura como codigo hereda los mismos problemas del desarrollo de software: el codigo puede contener errores de sintaxis, logicos o de seguridad que no se manifiestan hasta el momento de la ejecucion. La diferencia es que un error en Terraform puede traducirse en la destruccion de recursos en produccion, no en una excepcion en tiempo de compilacion. Por eso la validacion y el testing no son opcionales, sino parte integral del flujo de trabajo.

## terraform validate: verificacion sintactica y estructural

El comando `terraform validate` comprueba que la configuracion es sintacticamente valida y que las referencias entre recursos son coherentes. No necesita conexion con el proveedor cloud ni con el backend remoto, por lo que es la primera linea de defensa y la mas rapida:

```bash
terraform init
terraform validate
```

`validate` detecta errores como referencias a recursos inexistentes, tipos de datos incorrectos en variables, o bloques mal formados. Es el equivalente funcional de un linter estructural. Debe ejecutarse en cada cambio antes de llegar al `plan`, idealmente en un hook de pre-commit o en el primer paso de la pipeline de CI.

El comando admite el flag `-json` para integrar la salida con herramientas de analisis automatico, y puede limitarse a un directorio especifico con `-target`. Un `validate` que pasa no garantiza que la configuracion sea correcta semanticamente, pero asegura que Terraform puede cargar y analizar todos los ficheros sin errores de parseo.

## terraform plan: simulacion sin efectos secundarios

`terraform plan` va un paso mas alla: conecta con el backend remoto, lee el state actual, evalua la configuracion contra los providers y genera un conjunto de cambios propuestos. No modifica ningun recurso real, pero muestra exactamente que se creara, modificara o destruira:

```bash
terraform plan -out=tfplan
terraform show tfplan
```

El modo `-out` guarda el plan en un fichero binario que puede aplicarse posteriormente con `terraform apply tfplan`, garantizando que el apply ejecuta exactamente los cambios revisados. En entornos CI/CD, el plan debe revisarse manualmente o compararse contra reglas automatizadas antes de proceder al apply.

Para entornos de multiples equipos, `terraform plan` combinado con politicas de Sentinel (en Terraform Cloud) o con scripts de verificacion externos permite bloquear cambios que violen restricciones de seguridad o de coste. Un plan que crea un security group con puerto 22 abierto a `0.0.0.0/0` debe fallar automaticamente en la pipeline, no en el code review.

> El plan solo es fiable si el state remoto esta actualizado. Un `terraform refresh` previo o un plan con `-refresh=true` (comportamiento por defecto) garantiza que la simulacion parte del estado real de la infraestructura.

## tflint: linting especifico de Terraform

`tflint` es un linter disenado exclusivamente para Terraform que va mucho mas alla de `validate`. Detecta issues como el uso de instancias EC2 de tipo deprecated, la omision de tags obligatorios, o configuraciones de seguridad recomendadas que no se estan aplicando:

```bash
tflint --format=checkstyle
```

`tflint` trabaja con reglas modulares: incluye reglas por defecto para AWS, Azure y GCP, y permite anadir reglas personalizadas o desactivar las que no apliquen al proyecto. Su integracion en CI es directa, y el formato `checkstyle` es compatible con la mayoria de plataformas de analisis estatico (SonarQube, Jenkins, GitLab Code Quality).

Un uso tipico es forzar que todos los recursos tengan el tag `Environment` y `Owner`, o que los buckets S3 tengan el bloqueo de acceso publico habilitado. `tflint` no solo senala el problema, sino que en muchos casos ofrece la correccion automatica mediante `--fix`.

## tfsec: analisis de seguridad estatico

`tfsec` (ahora `trivy` en su modulo de Terraform) escanea la configuracion en busca de vulnerabilidades de seguridad conocidas. Mientras que `tflint` se centra en estilo y buenas practicas generales, `tfsec` evalua cada recurso contra un catalogo de malas practicas de seguridad:

```bash
tfsec .
```

Las reglas de `tfsec` cubren desde puertos SSH abiertos a internet hasta buckets S3 sin encriptacion, pasando por politicas de IAM demasiado permisivas o contrasenas hardcodeadas en variables. Cada hallazgo incluye una referencia a la guia de mitigacion y una severidad (CRITICAL, HIGH, MEDIUM, LOW).

La integracion en CI es fundamental: un hallazgo de severidad CRITICAL debe bloquear el merge de la Pull Request. `tfsec` soporta el formato SARIF para integrarse con GitHub Advanced Security y mostrar los resultados directamente en la pestana de seguridad del repositorio.

```bash
tfsec --format=sarif --out=results.sarif .
```

## Terratest: testing de integracion en Go

El testing de infraestructura no termina con el analisis estatico. `Terratest` es una libreria de Go que permite escribir tests de integracion que ejecutan Terraform, aplican la configuracion y verifican que los recursos creados se comportan como se espera:

```go
package test

import (
    "testing"
    "github.com/gruntwork-io/terratest/modules/terraform"
    "github.com/stretchr/testify/assert"
)

func TestVPCDeployment(t *testing.T) {
    terraformOptions := &terraform.Options{
        TerraformDir: "../examples/vpc",
    }

    defer terraform.Destroy(t, terraformOptions)
    terraform.InitAndApply(t, terraformOptions)

    vpcID := terraform.Output(t, terraformOptions, "vpc_id")
    assert.NotEmpty(t, vpcID)
}
```

El patron basico de Terratest es: `InitAndApply`, verificar outputs con assert, y `Destroy` en el `defer` para asegurar limpieza incluso si el test falla. Los tests pueden validar que una instancia responde en un puerto concreto, que un balanceador distribuye trafico, o que un bucket contiene los objetos esperados.

Terratest es especialmente util para validar modulos reutilizables. Un modulo que se publica en el registry de Terraform debe tener tests que verifiquen su comportamiento en diferentes combinaciones de variables. Sin testing de integracion, un cambio en el modulo puede romper a todos los consumidores sin que nadie lo detecte hasta el proximo apply.

Para entornos donde Go no es viable, alternativas como `terragrunt validate-all` con hooks personalizados o pytest-terraform (Python) ofrecen enfoques similares con diferentes niveles de madurez.

## Jerarquia de validacion

La combinacion de todas estas herramientas forma una piramide de validacion:

1. **Pre-commit**: `terraform fmt`, `terraform validate`, `tflint`, `tfsec`
2. **CI (pull request)**: `terraform plan` con revision automatizada, `tfsec` bloqueante
3. **CI (merge a main)**: `terraform apply` solo si el plan ha sido aprobado
4. **Periodico**: Terratest sobre los modulos principales para detectar regresiones

Cada capa captura una categoria distinta de errores: sintaxis, estilo, seguridad, simulacion y comportamiento real. Una pipeline que omite cualquiera de estas capas esta aceptando un riesgo que podria haberse detectado antes de llegar a produccion.

