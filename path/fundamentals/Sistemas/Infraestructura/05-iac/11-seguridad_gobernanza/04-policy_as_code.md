# Policy as Code

Policy as Code es la aplicacion del paradigma de infraestructura como codigo a las politicas de gobernanza y cumplimiento. En lugar de que las reglas de seguridad y organizacionales existan como documentos PDF, wiki pages o verificaciones manuales, se expresan como codigo ejecutable que se valida automaticamente durante el ciclo de vida del despliegue. Esto convierte las politicas de "deberia" en "debe", porque el pipeline de CI/CD rechaza cualquier configuracion que no cumpla con las reglas definidas.

La motivacion fundamental es la velocidad sin sacrificar control. En equipos grandes, un revisor humano no puede inspeccionar manualmente cada recurso nuevo para verificar que tenga las tags correctas, que no exponga puertos publicos innecesarios o que use tipos de instancia dentro del presupuesto aprobado. Las politicas como codigo ejecutan estas verificaciones en segundos, de forma consistente y sin excepciones.

## Open Policy Agent y Conftest

OPA es un motor de politicas generalista que usa el lenguaje Rego para definir reglas. Conftest es un wrapper que aplica OPA especificamente a archivos de configuracion como Terraform plan JSON, Kubernetes manifests, Dockerfiles o CloudFormation templates.

```rego
# policies/deny_public_s3.rego
package main

deny[msg] {
  resource := input.resource.aws_s3_bucket[_]
  not resource.acl
  msg := sprintf("Bucket %s no tiene ACL definido explicitamente", [resource._terraform.address])
}

deny[msg] {
  resource := input.resource.aws_s3_bucket[_]
  resource.acl == "public-read"
  msg := sprintf("Bucket %s tiene ACL public-read, uso prohibido", [resource._terraform.address])
}

deny[msg] {
  resource := input.resource.aws_s3_bucket[_]
  not resource.server_side_encryption_configuration
  msg := sprintf("Bucket %s no tiene cifrado habilitado", [resource._terraform.address])
}
```

Para ejecutar las politicas contra el plan de Terraform:

```bash
# Generar el plan en formato JSON
terraform plan -out=tfplan
terraform show -json tfplan > tfplan.json

# Ejecutar las politicas de OPA contra el plan
conftest test tfplan.json -p policies/ --all-namespaces
```

> Conftest retorna exit code distinto de cero si alguna politica falla, lo cual detiene automaticamente el pipeline de CI/CD. Esto es la base del modelo "fail closed" donde lo predeterminado es rechazar.

## HashiCorp Sentinel

Sentinel es el framework de politicas integrado nativamente en HashiCorp Cloud Platform y Terraform Enterprise. A diferencia de OPA que es una herramienta externa, Sentinel se ejecuta dentro del propio flujo de Terraform, interceptando el plan antes de que se aplique. Esto lo hace ideal para organizaciones que ya usan el ecosistema HashiCorp.

```python
# policies/enforce_cost_tags.sentinel
import "tfplan/v2" as tfplan

required_tags = ["Environment", "CostCenter", "Owner", "ManagedBy"]

# Verificar que todas las instancias EC2 tengan las tags requeridas
ec2_instances = filter tfplan.resource_changes as _, rc {
    rc.type is "aws_instance" and
    rc.mode is "managed" and
    (rc.actions contains "create" or rc.actions contains "update")
}

tag_violation = rule {
    all ec2_instances as _, instance {
        tags := instance.change.after.tags or {}
        all required_tags as tag {
            tags contains tag
        }
    }
}

# Verificar que los buckets S3 no sean publicos
s3_buckets = filter tfplan.resource_changes as _, rc {
    rc.type is "aws_s3_bucket" and
    rc.mode is "managed" and
    (rc.actions contains "create" or rc.actions contains "update")
}

s3_no_public = rule {
    all s3_buckets as _, bucket {
        bucket.change.after.acl != "public-read" and
        bucket.change.after.acl != "public-read-write"
    }
}

# Regla principal: todas las politicas deben pasar
main = rule {
    tag_violation and
    s3_no_public
}
```

> Sentinel se ejecuta en tres fases: plan (valida el plan), apply (valida el resultado real) y policy (reglas generales). Las politicas pueden ser "hard-mandatory" (imposible sobrepasar) o "soft-mandatory" (puede ser sobrepasado con autorizacion explicita).

## Checkov

Checkov es una herramienta de escaneo estatico desarrollada por Bridgecrew que analiza archivos IaC buscando problemas de seguridad y cumplimiento. A diferencia de Sentinel que se integra con el plan de Terraform, Checkov analiza directamente los archivos `.tf` fuente, lo que lo hace util como primer filtro antes de generar el plan.

```bash
# Escanear el directorio actual
checkov -d .

# Escanear solo archivos Terraform con output estructurado
checkov -d . --framework terraform --output json > checkov-results.json

# Ejecutar solo politicas especificas
checkov -d . --check CKV_AWS_18,CKV_AWS_19,CKV_AWS_20
```

Los resultados de Checkov incluyen el identificador de la politica, la severidad, el recurso afectado y la recomendacion:

```
Check: CKV_AWS_18: "Ensure the S3 bucket has access logging enabled"
	FAILED for resource: aws_s3_bucket.assets
	File: /modules/storage/main.tf:1-15
	Guide: https://docs.bridgecrew.io/docs/s3_3-enable-logging

Check: CKV_AWS_19: "Ensure the S3 bucket server side encryption is set to aws:kms"
	PASSED for resource: aws_s3_bucket.assets

Check: CKV_AWS_20: "Ensure the S3 bucket bucket policy is set to deny HTTP requests"
	FAILED for resource: aws_s3_bucket_policy.assets
	File: /modules/storage/main.tf:17-30
	Guide: https://docs.bridgecrew.io/docs/s3_4-data-in-transit
```

## Politicas de ejemplo

Una coleccion de politicas efectivas combina verificaciones de seguridad, cumplimiento normativo y restricciones de costos.

```python
# policies/require_private_subnets.sentinel
import "tfplan/v2" as tfplan

# Todas las subnets deben ser privadas excepto las designadas como publicas
subnets = filter tfplan.resource_changes as _, rc {
    rc.type is "aws_subnet" and
    rc.mode is "managed" and
    rc.actions contains "create"
}

required_cidr = rule {
    all subnets as _, subnet {
        cidr := subnet.change.after.cidr_block
        # Rangos RFC 1918 exclusivamente
        cidr matches "^(10\\.|172\\.(1[6-9]|2[0-9]|3[01])\\.|192\\.168\\.)"
    }
}
```

```python
# policies/limit_instance_sizes.sentinel
import "tfplan/v2" as tfplan

allowed_instance_types = [
    "t3.micro", "t3.small", "t3.medium",
    "m5.large", "m5.xlarge", "m5.2xlarge"
]

instances = filter tfplan.resource_changes as _, rc {
    rc.type is "aws_instance" and
    rc.mode is "managed" and
    rc.actions contains "create"
}

instance_within_budget = rule {
    all instances as _, instance {
        instance_type := instance.change.after.instance_type
        instance_type in allowed_instance_types
    }
}
```

| Herramienta | Integracion | Ventaja principal |
|-------------|-------------|-------------------|
| OPA/Conftest | CI/CD generico | Lenguaje Rego, multiproveedor |
| Sentinel | Terraform Enterprise/Cloud | Integracion nativa, hard/soft mandatory |
| Checkov | CI/CD generico | Base de politicas predefinida, multiproveedor |

