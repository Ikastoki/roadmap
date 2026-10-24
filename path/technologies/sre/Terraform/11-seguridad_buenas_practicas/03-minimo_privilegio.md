# Principio de minimo privilegio

El principio de minimo privilegio aplicado a Terraform consiste en disenar las credenciales y los roles de IAM de forma que cada identidad tenga exactamente los permisos necesarios para ejecutar las operaciones que le corresponden, y ni uno mas. En la practica, esto se traduce en tres decisiones concretas: que identidad ejecuta Terraform, que permisos tiene esa identidad, y durante cuanto tiempo son validos esos permisos.

## Credenciales con permisos justos

El error mas comun y peligroso es utilizar credenciales de administrador del proveedor cloud para ejecutar Terraform. Un par de access key y secret key de un usuario root o de un usuario con `AdministratorAccess` permite a Terraform crear cualquier recurso, pero tambien permite a un atacante que intercepte esas claves destruir toda la infraestructura.

Cada conjunto de credenciales debe estar vinculado a una politica que limite los servicios, las acciones y los recursos sobre los que Terraform puede operar. Por ejemplo, para un equipo que solo gestiona recursos de red en AWS, la politica debe permitir unicamente las acciones necesarias sobre VPC, subnets, security groups y route tables:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:CreateVpc",
        "ec2:DeleteVpc",
        "ec2:DescribeVpcs",
        "ec2:CreateSubnet",
        "ec2:DeleteSubnet",
        "ec2:CreateSecurityGroup",
        "ec2:AuthorizeSecurityGroupIngress"
      ],
      "Resource": "*"
    }
  ]
}
```

El uso de `Resource: "*"` es inevitable en muchos casos porque Terraform necesita seleccionar recursos por ID que se determinan en tiempo de ejecucion, pero las acciones deben acotarse estrictamente. Herramientas como `aws-iam-generator` o el analisis de logs de CloudTrail pueden ayudar a refinar la politica hasta el minimo necesario.

> Una politica demasiado permisiva es un riesgo silencioso: Terraform funciona correctamente, pero cualquier compromise de las credenciales tiene un radio de explosion maximo. Reducir permisos es siempre un ejercicio de validacion continua, no una configuracion inicial unica.

## IAM roles para entornos CI/CD

En pipelines de CI/CD, el uso de access keys de larga duracion es una practica que debe evitarse. En su lugar, el entorno de ejecucion (GitHub Actions, GitLab CI, Jenkins) debe asumir un rol de IAM mediante OIDC (OpenID Connect) o mediante SAML, de modo que las credenciales sean temporales y se roten automaticamente en cada ejecucion:

```hcl
data "aws_iam_policy_document" "github_actions_trust" {
  statement {
    effect  = "Allow"
    actions = ["sts:AssumeRoleWithWebIdentity"]
    principals {
      type        = "Federated"
      identifiers = ["arn:aws:iam::123456789012:oidc-provider/token.actions.githubusercontent.com"]
    }
    condition {
      test     = "StringEquals"
      variable = "token.actions.githubusercontent.com:sub"
      values   = ["repo:mi-organizacion/mi-repo:ref:refs/heads/main"]
    }
  }
}
```

Con esta configuracion, GitHub Actions puede asumir el rol sin necesidad de almacenar ningun secret key. El token OIDC lo proporciona GitHub automaticamente y es valido unicamente durante la ejecucion del workflow. El mismo patron se aplica a GitLab CI con su propio OIDC provider y a cualquier sistema compatible con federacion de identidades.

En GCP, el equivalente son las cuentas de servicio con Workload Identity Federation, que permite a los pipelines de CI intercambiar un token OIDC por un token de acceso de GCP sin necesidad de almacenar claves JSON de cuenta de servicio.

## Eliminacion de access keys de larga duracion

Las access keys de IAM que no expiran son un pasivo de seguridad. Una key filtrada a un repositorio publico, a un log de build o a un fichero de configuracion accidentalmente comiteado sigue siendo valida hasta que alguien la revoca manualmente. La rotacion automatica y las claves temporales eliminan esta ventana de exposicion.

La politica ideal es: ningun usuario humano utiliza access keys de IAM. Toda operacion de Terraform en produccion se ejecuta desde un rol asumido por una identidad federada (OIDC o SAML) o desde una instancia con un instance profile. Para operaciones locales de desarrollo, el desarrollador asume un rol mediante AWS CLI con `aws sts assume-role` o utiliza `gcloud auth application-default login` en GCP, que genera credenciales efimeras.

Si por restricciones tecnicas es inevitable usar access keys, deben rotarse cada 90 dias como maximo y marcarse como inactivas en cuanto se detecte que no se utilizan. Servicios como AWS IAM Access Analyzer pueden identificar claves no utilizadas y generar recomendaciones de eliminacion.

## Separacion de entornos por roles

Cada entorno (desarrollo, staging, produccion) debe tener su propio rol de IAM con permisos limitados a los recursos de ese entorno. Esto evita que un error en un pipeline de desarrollo pueda afectar a produccion, y limita el alcance de un posible compromise:

```hcl
resource "aws_iam_role" "terraform_apply" {
  name = "terraform-apply-${var.environment}"

  assume_role_policy = data.aws_iam_policy_document.trust.json
}

resource "aws_iam_role_policy" "terraform_apply" {
  name = "terraform-apply-policy-${var.environment}"
  role = aws_iam_role.terraform_apply.id

  policy = templatefile("policies/${var.environment}.json", {
    environment = var.environment
  })
}
```

La separacion por roles refuerza ademas la trazabilidad: cada operacion queda registrada en CloudTrail con la identidad del rol asumido, el entorno y la accion ejecutada. Sin esta separacion, todas las operaciones aparecen bajo la misma identidad y es imposible determinar si un cambio proviene de desarrollo o de produccion.

## Buenas practicas

El minimo privilegio no es un estado alcanzable de una sola vez, sino un proceso iterativo. Cada vez que se anade un recurso nuevo a la configuracion, la politica debe ampliarse con las acciones necesarias. La revision periodica de los permisos concedidos y la eliminacion de aquellos que ya no se corresponden con los recursos gestionados es la unica forma de mantener el principio a largo plazo. La combinacion de roles OIDC en CI/CD, credenciales temporales y politicas especificas por entorno reduce la superficie de ataque al minimo teorico posible.

