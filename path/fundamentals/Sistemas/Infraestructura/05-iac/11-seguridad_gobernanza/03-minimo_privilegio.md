# Principio de minimo privilegio

El principio de minimo privilegio establece que cada entidad, ya sea un usuario, un servicio o un proceso, debe tener exclusivamente los permisos necesarios para realizar su tarea especifica y nada mas. En el contexto de infraestructura como codigo, este principio se aplica de forma bidireccional: las credenciales que Terraform o cualquier herramienta de IaC utilizan para interactuar con proveedores cloud deben estar estrictamente limitadas, y los recursos provisionados deben seguir la misma restriccion en sus politicas de acceso.

La consecuencia mas comun de ignorar este principio es el escalamiento de privilegios. Si el rol de Terraform tiene permisos de administrador en la cuenta AWS, una vulnerabilidad en cualquier modulo, un error en un script o un ataque al pipeline de CI/CD resulta en compromiso total de la cuenta. En su lugar, un rol con permisos limitados a los servicios especificos que gestiona contiene el impacto de cualquier incidente al perimeter de esos recursos.

## Rol IAM para Terraform

En lugar de usar credenciales de usuario IAM (access keys) que son estaticas y de larga duracion, la practica recomendada consiste en asumir un IAM Role temporal desde el pipeline de CI/CD. Esto elimina la necesidad de rotar credenciales manualmente y permite que cada entorno (dev, staging, production) tenga su propio rol con permisos ajustados.

```hcl
resource "aws_iam_role" "terraform_execution" {
  name = "terraform-execution-${var.environment}"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = {
          AWS = "arn:aws:iam::123456789012:role/github-actions-deployer"
        }
        Action = "sts:AssumeRole"
      }
    ]
  })
}
```

> Usar `sts:AssumeRole` en lugar de access keys estaticas permite que AWS valide la identidad del caller, registre quien asumio el rol, y aplique las conditions de session tags para politicas mas granulares.

## Politica de minimo privilegio

La politica adjunta al rol de Terraform debe listar explicitamente los servicios y acciones necesarias. No se debe usar `*` en Resource ni en Action, y cada servicio debe restringirse a los ARN especificos que Terraform necesita gestionar.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowEC2Management",
      "Effect": "Allow",
      "Action": [
        "ec2:RunInstances",
        "ec2:TerminateInstances",
        "ec2:DescribeInstances",
        "ec2:DescribeInstanceStatus",
        "ec2:CreateTags",
        "ec2:CreateSecurityGroup",
        "ec2:AuthorizeSecurityGroupIngress",
        "ec2:AuthorizeSecurityGroupEgress",
        "ec2:DeleteSecurityGroup",
        "ec2:ModifyInstanceAttribute",
        "ec2:DescribeSubnets",
        "ec2:DescribeVpcs",
        "ec2:DescribeSecurityGroups"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "aws:RequestedRegion": "eu-west-1"
        }
      }
    },
    {
      "Sid": "AllowS3ForStateBackend",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::company-terraform-state-*",
        "arn:aws:s3:::company-terraform-state-*/*"
      ]
    },
    {
      "Sid": "AllowDynamoDBForStateLock",
      "Effect": "Allow",
      "Action": [
        "dynamodb:GetItem",
        "dynamodb:PutItem",
        "dynamodb:DeleteItem"
      ],
      "Resource": "arn:aws:dynamodb:eu-west-1:123456789012:table/terraform-state-lock"
    },
    {
      "Sid": "AllowIAMReadOnlyForDataSource",
      "Effect": "Allow",
      "Action": [
        "iam:GetRole",
        "iam:GetRolePolicy",
        "iam:ListRoles",
        "iam:ListInstanceProfiles"
      ],
      "Resource": "*"
    },
    {
      "Sid": "DenyDangerousActions",
      "Effect": "Deny",
      "Action": [
        "iam:CreateUser",
        "iam:CreateAccessKey",
        "iam:DeleteRole",
        "organizations:*",
        "account:*"
      ],
      "Resource": "*"
    }
  ]
}
```

> La condicion `aws:RequestedRegion` en el bloque de EC2 evita que Terraform provisione recursos fuera de la region autorizada. Esto es particularmente importante cuando se trabaja con multiples cuentas o cuando los costos de la region deben controlarse.

## Estrategia de politicas por entorno

Un patron efectivo consiste en definir una politica base comun y capas adicionales de permisos segun el entorno. El entorno de desarrollo obtiene permisos amplios para experimentar, staging replica la produccion con restricciones menores, y produccion es estrictamente limitado a los recursos existentes.

```hcl
module "terraform_role" {
  source = "./modules/terraform-iam-role"

  environment = var.environment
  
  permissions_boundary = "arn:aws:iam::123456789012:policy/terraform-permissions-boundary"
  
  extra_policies = var.environment == "prod" ? [
    aws_iam_policy.s3_readonly.arn,
    aws_iam_policy.dynamodb_readonly.arn
  ] : [
    aws_iam_policy.full_access.arn
  ]
}

resource "aws_iam_policy" "permissions_boundary" {
  name   = "terraform-permissions-boundary"
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Deny"
        Action = [
          "iam:*",
          "organizations:*",
          "account:*",
          "s3:DeleteBucket"
        ]
        Resource = "*"
      }
    ]
  })
}
```

> Un `permissions_boundary` actua como techo maximo de permisos. Incluso si un modulo o politica adjunta concede permisos adicionales, el boundary los limita. Es una capa de defensa que protege contra configuraciones excesivas.

## Revision y monitoreo de permisos

Los permisos no son estaticos. A medida que la infraestructura evoluciona, los roles pueden acumular permisos innecesarios o quedar desactualizados. AWS Access Analyzer detecta politicas con permisos excesivos y sugiere politicas restringidas. IAM Access Analyzer tambien identifica recursos compartidos publicamente o con cuentas externas.

```bash
# Analizar politicas para detectar permisos excesivos
aws accessanalyzer start-policy-generation \
  --cloud-trail-details file://cloudtrail-details.json \
  --policy-generation-details file://policy-details.json

# Ver resultados del analisis
aws accessanalyzer get-generated-policy \
  --job-id 12345678-1234-1234-1234-123456789012 \
  --include-resource-placeholders
```

| Practica | Implementacion |
|----------|----------------|
| Access keys estaticas | Prohibidas, usar ST AssumeRole |
| Permisos de region | Condicion `aws:RequestedRegion` |
| Permisos Boundary | Techo maximo en todos los roles |
| Revision periodica | IAM Access Analyzer cada 30 dias |
| Eliminacion de permisos | Politica de vida util para roles temporales |

