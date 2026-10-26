# Gestion de secretos en IaC

La gestion de secretos constituye una de las areas mas criticas cuando se trabaja con infraestructura como codigo. Cualquier archivo que defina recursos de infraestructura inevitablemente necesita interactuar con credenciales: tokens de API, contrasenas de bases de datos, certificados TLS, claves de cifrado y credenciales de servicios cloud. Si alguno de estos valores queda almacenado en texto plano dentro del repositorio de codigo fuente, se convierte en un vector de ataque de alto impacto: cualquier persona con acceso al historial de git, ya sea un desarrollador, un ex-empleado o un atacante que comprometa el repositorio, obtiene acceso a las credenciales de toda la infraestructura. El principio fundamental es que los secretos nunca deben residir en el codigo fuente, ni en archivos de configuracion versionados, ni en variables de entorno hardcodeadas.

La solucion arquitectonica consiste en integrar un gestor externo de secretos que actue como fuente unica de verdad para todas las credenciales. Herramientas como HashiCorp Vault, AWS Secrets Manager, Azure Key Vault o Google Cloud Secret Manager almacenan los secretos de forma cifrada y proporcionan mecanismos de acceso controlado, rotacion automatica y auditoria completa de quien accede a que secreto y cuando. Desde la perspectiva de IaC, estos gestores se integran de dos formas principales: como Data Sources que leen el valor del secreto en tiempo de plan/apply, o como inyeccion dinamica en runtime cuando el recurso provisionado lo necesita.

## Marcar valores como sensibles

Terraform dispone del atributo `sensitive` en variables y outputs que impide que los valores se muestren en la salida de la CLI. Esto no cifra el valor ni lo protege en el estado, pero evita la exposicion accidental en logs, terminal o en la documentacion de outputs.

```hcl
variable "db_password" {
  description = "Contrasena de la base de datos principal"
  type      = string
  sensitive = true
}

output "db_endpoint" {
  value     = aws_db_instance.primary.endpoint
  sensitive = true
}
```

> `sensitive = true` solo oculta el valor en la salida de Terraform. El valor sigue en texto plano dentro del state file. Por eso la proteccion del state file es igualmente critica.

## Integracion con HashiCorp Vault

Vault permite leer secretos desde diferentes motores (KV, database, AWS, PKI) y Terraform los consume mediante el provider `vault`. El flujo comienza configurando la autenticacion del provider, luego se leen los secretos y finalmente se usan en los recursos correspondientes.

```hcl
provider "vault" {
  address = "https://vault.example.com:8200"
}

data "vault_generic_secret" "db_credentials" {
  path = "secret/data/production/database"
}

resource "aws_db_instance" "primary" {
  identifier           = "prod-db-01"
  engine               = "postgres"
  instance_class       = "db.r6g.large"
  allocated_storage    = 100
  username             = data.vault_generic_secret.db_credentials.data["username"]
  password             = data.vault_generic_secret.db_credentials.data["password"]
  skip_final_snapshot  = true
}
```

Vault ofrece un mecanismo adicional llamado Dynamic Secrets que genera credenciales temporales con un TTL limitado. En lugar de leer una contrasena estatica, Vault crea un usuario efimero en la base de datos con permisos exactos y lo elimina automaticamente cuando expira. Esto reduce drasticamente la superficie de ataque.

## Integracion con AWS Secrets Manager

En entornos completamente gestionados dentro de AWS, Secrets Manager es la alternativa nativa. Terraform lo consulta como Data Source sin necesidad de un provider adicional si se usa el provider AWS.

```hcl
data "aws_secretsmanager_secret_version" "db_password" {
  secret_id = "prod/database/master-password"
}

resource "aws_db_instance" "primary" {
  identifier      = "prod-db-01"
  engine          = "postgres"
  instance_class  = "db.r6g.large"
  allocated_storage = 100
  username        = "admin"
  password        = data.aws_secretsmanager_secret_version.db_password.secret_string
}
```

> AWS Secrets Manager permite configurar rotacion automatica mediante AWS Lambda cada N dias. Cuando se usa con Terraform, el ciclo de vida del secreto se gestiona por completo fuera del repositorio de codigo.

## Inyeccion de secretos en runtime

Un patron avanzado consiste en que Terraform provisiona la infraestructura pero los secretos nunca pasan por Terraform. En su lugar, la maquina o contenedor que ejecuta la aplicacion obtiene los secretos directamente del gestor en tiempo de arranque. Esto es comun en entornos Kubernetes donde los secrets se inyectan como variables de entorno o volumenes montados.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-backend
spec:
  template:
    spec:
      containers:
        - name: backend
          env:
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: password
```

Este patron es preferible cuando la aplicacion tiene un ciclo de vida propio y no necesita que los secretos esten disponibles durante el provisioning de infraestructura.

## Reglas de seguridad para secretos en IaC

| Regla | Descripcion |
|-------|-------------|
| Nunca en git | Los secretos no se versionan, ni siquiera cifrados si la clave esta en el mismo repositorio |
| State protegido | El state contiene secretos en texto plano, requiere cifrado y acceso restringido |
| Rotacion automatica | Los secretos se rotan periodicamente y las aplicaciones los refrescan sin reinicio |
| Auditoria | Cada lectura de un secreto queda registrada con timestamp, usuario y origen |
|最小 permisos | Los roles de acceso a secretos solo leen los paths que necesitan |

