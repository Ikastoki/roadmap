# State remoto seguro

El state file de Terraform contiene el mapa completo de recursos gestionados: IDs de instancias, direcciones IP, configuraciones de recursos y, a menos que se hayan tomado medidas explicitas, valores de secretos en texto plano. Proteger este fichero no es un requisito operativo menor, sino una condicion indispensable para considerar un despliegue de Terraform como seguro. Un state file filtrado entrega a un atacante un inventario detallado de la infraestructura y, potencialmente, credenciales de acceso.

## Encriptacion en reposo en S3

Cuando se utiliza AWS S3 como backend remoto, la encriptacion del state file debe configurarse explicitamente. S3 ofrece encriptacion del lado del servidor (SSE) con claves gestionadas por AWS (SSE-S3), con claves de KMS (SSE-KMS) o con claves propias (SSE-C). La opcion recomendada es SSE-KMS, ya que permite auditar el acceso al state mediante CloudTrail y aplicar politicas de rotacion:

```hcl
terraform {
  backend "s3" {
    bucket         = "terraform-state-produccion"
    key            = "infraestructura/terraform.tfstate"
    region         = "eu-west-1"
    encrypt        = true
    kms_key_id     = "alias/terraform-state-key"
    dynamodb_table = "terraform-state-locks"
  }
}
```

El argumento `encrypt = true` habilita SSE-S3. Para usar KMS es necesario especificar `kms_key_id` y la politica de IAM del bucket debe permitir `kms:Decrypt` a los roles autorizados. Sin encriptacion, cualquier persona con acceso de lectura al bucket obtiene el contenido completo del state.

## Encriptacion en GCS

Google Cloud Storage admite encriptacion por defecto con claves gestionadas por Google, pero para un control granular se recomienda utilizar Cloud KMS y una clave propia. La configuracion del backend en GCS requiere la clave de encriptacion referenciada por su identificador de recurso:

```hcl
terraform {
  backend "gcs" {
    bucket         = "terraform-state-produccion"
    prefix         = "infraestructura"
    encryption_key = "projects/mi-proyecto/locations/global/keyRings/tf-state/cryptoKeys/state-key"
  }
}
```

A diferencia de S3, GCS aplica encriptacion automatica en reposo sin necesidad de un flag adicional, pero el uso de Cloud KMS permite controlar quien puede desencriptar el state mediante permisos IAM sobre la clave. Sin una clave dedicada, cualquier principal con acceso al bucket puede leer el state, aunque Google gestione la capa de cifrado subyacente.

## Control de acceso IAM estricto

La encriptacion protege el fichero en reposo, pero no controla quien puede leerlo. La politica de acceso sobre el bucket de state debe seguir el principio de minimo privilegio: solo los roles o cuentas de servicio que ejecutan Terraform (ya sea en local, en un runner de CI/CD o en Terraform Cloud) deben tener permisos de lectura y escritura sobre el state.

En AWS, la politica del bucket S3 debe otorgar `s3:GetObject` y `s3:PutObject` exclusivamente a los principals autorizados, y denegar explicitamente el acceso publico mediante un bloque `public_access_block`:

```hcl
resource "aws_s3_bucket_public_access_block" "state" {
  bucket = aws_s3_bucket.state.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
```

En GCP, el bucket debe tener ACLs deshabilitadas y utilizar IAM con condiciones para restringir el acceso por origen de red o por identidad. La cuenta de servicio que ejecuta Terraform debe tener unicamente `roles/storage.objectAdmin` sobre el bucket, y ningun otro permiso a nivel de proyecto.

> Un error frecuente es dar permisos de `s3:ListBucket` o `storage.buckets.get` a roles amplios. Aunque parezcan inofensivos, permiten enumerar los ficheros de state y, combinados con otros permisos, filtrar informacion sobre el numero de entornos o el tamano de la infraestructura.

## State locking para consistencia

Cuando multiples personas o procesos ejecutan Terraform sobre el mismo state, se produce el riesgo de escrituras concurrentes que corrompen el fichero. El state locking evita esta condicion de carrera creando un bloqueo durante la ejecucion de `apply` y liberandolo al finalizar.

En S3, el locking se implementa mediante DynamoDB. Terraform adquiere un lock creando un item en la tabla de DynamoDB configurada; si otro proceso intenta adquirirlo, Terraform espera o falla segun la configuracion:

```hcl
resource "aws_dynamodb_table" "terraform_locks" {
  name         = "terraform-state-locks"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }
}
```

En GCS, el locking es nativo del backend: el propio servicio GCS garantiza la consistencia mediante operaciones atomicas en el objeto de state, sin necesidad de un recurso adicional. En Azure, el backend de `azurerm` utiliza el blob storage de Azure con soporte de leasing para implementar el bloqueo.

Sin state locking, dos ejecuciones simultaneas pueden sobreescribirse mutuamente, provocando perdida de recursos o divergencia entre el state y la infraestructura real. En entornos CI/CD donde multiples pipelines pueden ejecutarse en paralelo, el locking es obligatorio.

## Resumen de configuracion

Un backend remoto seguro combina cuatro capas: encriptacion en reposo (SSE-KMS o Cloud KMS), control de acceso IAM restrictivo, state locking (DynamoDB o leasing nativo) y politicas de bloqueo de acceso publico. Cualquiera de estas capas omitida convierte el state file en un punto de fallo critico para la seguridad de toda la infraestructura declarada.

