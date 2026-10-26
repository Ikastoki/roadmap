# Seguridad del state file

El state file de Terraform es el archivo mas sensible dentro de cualquier proyecto de infraestructura como codigo. Contiene un mapeo completo entre los recursos definidos en el codigo y los recursos reales desplegados en la infraestructura, incluyendo sus identificadores, atributos y valores actuales. Criticamente, el state almacena en texto plano valores que se marcaron como `sensitive`, como contrasenas de bases de datos, tokens de acceso y claves de cifrado. Esto significa que cualquiera que obtenga acceso al state file tiene un inventario completo de la infraestructura y las credenciales necesarias para acceder a ella.

El state file local por defecto se almacena en el disco del desarrollador, lo cual es inaceptable en cualquier entorno que no sea una demostracion. En produccion, el state se almacena en backends remotos como S3, Azure Blob Storage o GCS, que proporcionan cifrado en reposo, control de acceso basado en identidades y capacidad de bloqueo para operaciones concurrentes. La combinacion de cifrado, permisos estrictos y bloqueo de estado constituye la defensa en profundidad necesaria para proteger esta informacion critica.

## Backend remoto con S3 y DynamoDB

La configuracion mas comun en AWS combina S3 para el almacenamiento del state con DynamoDB para el bloqueo de estado. S3 proporciona versionado, cifrado y lifecycle policies, mientras DynamoDB garantiza que solo una operacion de Terraform pueda ejecutarse a la vez sobre un conjunto de recursos determinado.

```hcl
terraform {
  backend "s3" {
    bucket         = "company-terraform-state-prod"
    key            = "infrastructure/network/terraform.tfstate"
    region         = "eu-west-1"
    encrypt        = true
    dynamodb_table = "terraform-state-lock"
  }
}
```

> El `key` del state se usa como namespace logico. Estructurar la ruta por capa (`infrastructure/network/terraform.tfstate`) permite un desglose granular de permisos y una mejor organizacion cuando multiples equipos comparten el mismo bucket.

## Cifrado del bucket S3

El cifrado en reposo se configura en el bucket S3 y garantiza que los archivos del state se cifren antes de escribirse en disco. AWS KMS gestiona las claves de cifrado y permite rotacion automatica sin intervención manual.

```hcl
resource "aws_s3_bucket" "terraform_state" {
  bucket = "company-terraform-state-prod"

  tags = {
    Purpose = "terraform-state"
  }
}

resource "aws_s3_bucket_versioning" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id

  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"
      kms_master_key_id = aws_kms_key.terraform_state.arn
    }
    bucket_key_enabled = true
  }
}

resource "aws_s3_bucket_public_access_block" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

resource "aws_kms_key" "terraform_state" {
  description             = "Clave KMS para cifrado del state de Terraform"
  deletion_window_in_days = 30
  enable_key_rotation     = true
}
```

> `block_public_acls` y `restrict_public_buckets` son obligatorios para prevenir que un error de configuracion exponga el state publicamente. Incluso con cifrado activo, el acceso publico a un state file es un incidente de seguridad reportable.

## Politica de acceso al bucket

El control de acceso al bucket de state debe ser lo mas restrictivo posible. Solo los roles de CI/CD y los usuarios que ejecutan Terraform directamente necesitan acceso. La politica IAM se aplica a nivel de bucket y se complementa con las politicas de la cuenta o region.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowTerraformRoleAccess",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:role/terraform-execution-role"
      },
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject"
      ],
      "Resource": "arn:aws:s3:::company-terraform-state-prod/*"
    },
    {
      "Sid": "AllowDynamoDBLocking",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:role/terraform-execution-role"
      },
      "Action": [
        "dynamodb:GetItem",
        "dynamodb:PutItem",
        "dynamodb:DeleteItem"
      ],
      "Resource": "arn:aws:dynamodb:eu-west-1:123456789012:table/terraform-state-lock"
    }
  ]
}
```

> La politica anterior no incluye permisos de `s3:ListBucket` ni `s3:DeleteBucket` para el propio bucket. Esto previene que un error en el codigo elimine accidentalmente el bucket completo y con el todo su historial de states.

## Bloqueo de estado con DynamoDB

El bloqueo de estado previene condiciones de carrera cuando multiples personas o pipelines ejecutan Terraform simultaneamente sobre los mismos recursos. DynamoDB actua como un lock distribuido: antes de ejecutar cualquier operacion, Terraform intenta crear un registro en la tabla; si el registro ya existe, la operacion falla con un mensaje indicando quien tiene el lock activo.

```hcl
resource "aws_dynamodb_table" "terraform_lock" {
  name         = "terraform-state-lock"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }
}
```

En entornos donde DynamoDB no esta disponible, Terraform soporta bloqeo nativo en backends como Consul o S3 (habilitado con `dynamodb_table` o con la configuracion interna de S3 locking).

## Retencion de versiones y eliminacion

El versionado de S3 permite recuperar versions anteriores del state si un apply corrupto el archivo o si es necesario revertir a un estado conocido. Combinado con lifecycle policies, se puede retener un numero fijo de versiones y eliminar las mas antiguas para controlar costos.

```hcl
resource "aws_s3_bucket_lifecycle_configuration" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id

  rule {
    id     = "retain-versions"
    status = "Enabled"

    noncurrent_version_expiration {
      noncurrent_days = 90
    }

    noncurrent_version_transition {
      noncurrent_days = 30
      storage_class   = "STANDARD_IA"
    }
  }
}
```

