# Pruebas en infraestructura como codigo

Las pruebas en IaC difieren fundamentalmente de las pruebas de software tradicionales porque el sujeto bajo prueba no es logica de negocio sino recursos de infraestructura reales que tienen costo, estado y consecuencias materiales. Un test unitario de una funcion pura ejecuta en milisegundos y no tiene efectos secundarios; un test de infraestructura despliega servidores, bases de datos y redes que tardan minutos en estar listos y generan costos en la cuenta cloud. Esta realidad exige una estrategia de testing escalonada donde las verificaciones rapidas y economicas se ejecutan frecuentemente y las pruebas de integracion completas se ejecutan de forma selectiva.

La piramide de testing en IaC se estructura en tres niveles. Los estaticos analizan el codigo fuente sin ejecutarlo, detectando errores de sintaxis, best practices y vulnerabilidades conocidas. Los de integracion despliegan la infraestructura en un entorno real temporal, verifican que los recursos existan y funcionen correctamente, y finalmente destruyen todo para no dejar residuos. Los end-to-end validan la infraestructura desde la perspectiva del usuario final o del sistema que la consume.

## Pruebas estaticas con tflint

tflint es un linter para Terraform que verifica el codigo fuente contra reglas de estilo, deprecaciones y mejores practicas. No ejecuta Terraform ni despliega nada, por lo que es instantaneo y seguro de ejecutar en cualquier entorno.

```bash
# Instalar tflint
curl -s https://raw.githubusercontent.com/terraform-linters/tflint/master/install_linux.sh | bash

# Ejecutar en el directorio actual
tflint

# Ejecutar con configuracion personalizada
tflint --config .tflint.hcl
```

La configuracion de tflint se define en un archivo `.tflint.hcl` que especifica los plugins y reglas a aplicar:

```hcl
# .tflint.hcl
plugin "aws" {
  enabled = true
  version = "0.28.0"
  source  = "github.com/terraform-linters/tflint-ruleset-aws"
}

rule "terraform_naming_convention" {
  enabled = true
  format  = "snake_case"
}

rule "terraform_documented_variables" {
  enabled = true
}

rule "terraform_documented_outputs" {
  enabled = true
}

rule "terraform_unused_declarations" {
  enabled = true
}

# Reglas del plugin AWS
rule "aws_instance_invalid_type" {
  enabled = true
}

rule "aws_instance_previous_generation" {
  enabled = true
}

rule "aws_s3_bucket_name" {
  enabled = true
}
```

> tflint no reemplaza a `terraform validate`. Validate verifica la sintaxis Terraform y la coherencia del tipo de datos. tflint verifica contra un conjunto de reglas empiricas que reflejan lessons learned de la comunidad. Ambos se complementan.

## Pruebas estaticas con tfsec

tfsec analiza el codigo Terraform en busqueda de problemas de seguridad. A diferencia de tflint que se enfoca en calidad del codigo, tfsec se enfoca exclusivamente en configuraciones que representan riesgos de seguridad como buckets publicos, puertos abiertos o cifrado no habilitado.

```bash
# Ejecutar tfsec en el directorio actual
tfsec .

# Salida en formato JSON para integrar con pipelines
tfsec . --format json --out tfsec-results.json

# Excluir verificaciones especificas
tfsec . --exclude aws-s3-enable-bucket-logging
```

```
aws/s3/encryption_not_enabled: S3 bucket does not have encryption enabled
  /modules/storage/main.tf:1
    1 | resource "aws_s3_bucket" "assets" {
    2 |   bucket = "company-assets-prod"

aws/s3/block_public_acls: S3 bucket has public ACLs enabled
  /modules/storage/main.tf:8
    8 |   acl = "public-read"

aws/s3/enable_bucket_logging: S3 bucket does not have logging enabled
  /modules/storage/main.tf:1
    1 | resource "aws_s3_bucket" "assets" {
```

## Pruebas de integracion con Terratest

Terratest es un framework de pruebas escrito en Go que ejecuta el ciclo completo de Terraform (init, plan, apply, test, destroy) contra un entorno real. Las pruebas despliegan infraestructura, la verifican usando asserts, y la destruyen al final sin importar si la prueba paso o fallo. Esto garantiza que no quedan recursos huérfanos generando costos.

```go
package test

import (
    "testing"
    "github.com/gruntwork-io/terratest/modules/terraform"
    "github.com/gruntwork-io/terratest/modules/aws"
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
)

func TestVpcModule(t *testing.T) {
    t.Parallel()

    terraformOptions := terraform.WithDefaultRetryableErrors(t, &terraform.Options{
        TerraformDir: "../modules/vpc",
        Vars: map[string]interface{}{
            "environment": "test",
            "vpc_cidr":    "10.99.0.0/16",
        },
        NoColor: true,
    })

    // Garantizar destruccion al final
    defer terraform.Destroy(t, terraformOptions)

    // Desplegar la infraestructura
    terraform.InitAndApply(t, terraformOptions)

    // Obtener outputs
    vpcId := terraform.Output(t, terraformOptions, "vpc_id")
    publicSubnetIds := terraform.OutputList(t, terraformOptions, "public_subnet_ids")
    privateSubnetIds := terraform.OutputList(t, terraformOptions, "private_subnet_ids")

    // Verificar que el VPC existe en AWS
    vpc := aws.GetVpcById(t, vpcId, "eu-west-1")
    assert.Equal(t, "10.99.0.0/16", vpc.CidrBlock)

    // Verificar que las subnets existen y estan en AZs diferentes
    require.Len(t, publicSubnetIds, 2)
    require.Len(t, privateSubnetIds, 2)

    for _, subnetId := range publicSubnetIds {
        subnet := aws.GetSubnetById(t, subnetId, "eu-west-1")
        assert.True(t, subnet.AvailabilityZone != "", "Subnet must be in an AZ")
    }

    // Verificar que las subnets privadas no tienen IP publico
    for _, subnetId := range privateSubnetIds {
        subnet := aws.GetSubnetById(t, subnetId, "eu-west-1")
        assert.False(t, subnet.MapPublicIpOnLaunch,
            "Private subnet should not auto-assign public IPs")
    }
}
```

> `defer terraform.Destroy` garantiza que la infraestructura se destruya incluso si la prueba falla. Esto es fundamental para evitar costos acumulados por tests fallidos. En pipelines de CI, ejecutar con `-timeout 30m` previene que un apply atascado consuma tiempo infinito.

## Configuracion de Terratest

Un test de Terratest bien estructurado sigue el patron Arrange-Act-Assert con la variacion especifica de IaC: Arrange (configurar opciones de Terraform), Act (init + apply), Assert (verificar outputs y recursos reales), Cleanup (destroy).

```go
func TestDatabaseModule(t *testing.T) {
    t.Parallel()

    dbPassword := random.UniqueId()

    terraformOptions := terraform.WithDefaultRetryableErrors(t, &terraform.Options{
        TerraformDir: "../modules/database",
        Vars: map[string]interface{}{
            "environment":  "test",
            "db_name":      "testdb",
            "db_username":  "testadmin",
            "db_password":  dbPassword,
            "db_instance":  "db.t3.micro",
        },
        EnvVars: map[string]string{
            "AWS_DEFAULT_REGION": "eu-west-1",
        },
    })

    defer terraform.Destroy(t, terraformOptions)
    terraform.InitAndApply(t, terraformOptions)

    endpoint := terraform.Output(t, terraformOptions, "endpoint")
    port := terraform.Output(t, terraformOptions, "port")

    assert.NotEmpty(t, endpoint)
    assert.Equal(t, "5432", port)

    // Verificar que la instancia RDS existe
    dbInstance := aws.GetRdsInstance(t, endpoint, "eu-west-1")
    assert.Equal(t, "available", dbInstance.DBInstanceStatus)
    assert.Equal(t, "db.t3.micro", dbInstance.DBInstanceClass)
}
```

## Resumen de herramientas por tipo de prueba

| Tipo | Herramienta | Velocidad | Costo | Ejecucion |
|------|------------|-----------|-------|-----------|
| Estatico | tflint | Segundos | Gratis | Cada commit |
| Estatico | tfsec | Segundos | Gratis | Cada commit |
| Estatico | checkov | Segundos | Gratis | Cada commit |
| Integracion | Terratest | Minutos | Costo cloud | PR merge |
| Integracion | Kitchen-Terraform | Minutos | Costo cloud | PR merge |
| End-to-end | Custom scripts | Variable | Costo cloud | Release |

> Kitchen-Terraform sigue un patron similar a Terratest pero usa un framework de testing basado en Ruby (Test Kitchen) que gestiona el ciclo de vida de los "instances" (entornos temporales). Es una alternativa valida especialmente en equipos con experiencia en ecosistema Ruby.

