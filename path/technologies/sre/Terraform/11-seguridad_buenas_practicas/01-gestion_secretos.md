# Gestion de secretos

Un secreto incrustado en el codigo fuente es una vulnerabilidad con fecha de caducidad incierta pero con impacto garantizado. Cuando una cadena de conexion a base de datos, una clave de API o un token de servicio aparece literalmente en un fichero `.tf`, ese repositorio se convierte en un vector de ataque permanente: cualquier persona con acceso al historial de Git, a una copia del repositorio o a una pipeline de CI comprometida tiene credenciales validas. La disciplina de gestion de secretos en Terraform consiste exactamente en eso: eliminar cualquier valor sensible del codigo y sustituirlo por referencias a fuentes autorizadas que se resuelven en tiempo de ejecucion.

## Variables de entorno y el argumento sensitive

El mecanismo mas inmediato para evitar secretos literales es delegar su valor a variables de entorno. Terraform lee automaticamente variables con el prefijo `TF_VAR_` y las asigna a las variables de input correspondientes:

```hcl
variable "db_password" {
  description = "Contrasena de la base de datos"
  type        = string
  sensitive   = true
}
```

```bash
export TF_VAR_db_password="s3cur3P@ss"
terraform plan
```

El argumento `sensitive = true` tiene un efecto doble: impide que Terraform muestre el valor en la salida de `plan` o `apply` (lo sustituye por `(sensitive value)`) y, en Terraform Cloud, marca el valor como oculto en la interfaz web. Sin embargo, `sensitive` no es una medida de cifrado; el valor sigue almacenandose en texto plano en el state file a menos que se apliquen medidas adicionales.

> Un error comun es asumir que `sensitive` protege el secreto. Solo controla la visibilidad en la salida por terminal. El state file sigue conteniendo el valor en claro, por lo que proteger el state remoto es igual de prioritario.

## Terraform Cloud como backend de variables

Terraform Cloud ofrece un almacenamiento nativo de variables sensibles. En el flujo de trabajo remoto, las variables se definen en el workspace a traves de la interfaz web o la API, se marcan como "sensitive" y se inyectan en tiempo de ejecucion sin que el cliente las vea nunca. Esto elimina la necesidad de gestionar un fichero `terraform.tfvars` local con credenciales.

La configuracion del bloque `cloud` en el codigo declara el workspace, y las variables sensibles se configuran desde la consola de Terraform Cloud, no desde el repositorio:

```hcl
terraform {
  cloud {
    organization = "mi-org"
    workspaces {
      name = "produccion"
    }
  }
}
```

Las ventajas son claras: auditoria de acceso, rotacion centralizada y ausencia de secretos en el cliente. La contrapartida es la dependencia de un servicio externo y la necesidad de gestionar el token de acceso a Terraform Cloud, que si debe tratarse como un secreto de primer nivel.

## Vault como fuente de autoridad dinamica

HashiCorp Vault es el estandar para la gestion centralizada de secretos, y su integracion con Terraform se realiza mediante el provider de Vault. En lugar de leer una variable estatica, Terraform consulta a Vault en tiempo de ejecucion y obtiene el valor actual, que puede ser una credencial rotada, un token temporal o una clave con tiempo de vida limitado:

```hcl
data "vault_kv_secret_v2" "db" {
  mount = "kv"
  name  = "produccion/postgresql"
}

resource "aws_db_instance" "main" {
  username = data.vault_kv_secret_v2.db.data["username"]
  password = data.vault_kv_secret_v2.db.data["password"]
}
```

El provider de Vault se autentica mediante un token, que a su vez debe inyectarse como variable de entorno. Este patron elimina cualquier secreto del estado de Terraform si se combina con `sensitive = true` en el schema del data source, aunque el valor sigue presente transitoriamente en memoria durante la ejecucion.

Para entornos con politicas de rotacion agresivas, Vault permite generar credenciales de base de datos bajo demanda mediante sus secret engines de tipo `database`, de modo que Terraform obtiene una credencial efimera valida unicamente para el tiempo de provisionamiento.

## AWS Secrets Manager y Azure Key Vault

Cuando la infraestructura ya reside en un proveedor cloud, el servicio nativo de gestion de secretos evita introducir un salto adicional. AWS Secrets Manager y Azure Key Vault se integran con Terraform mediante data sources que recuperan el secreto en el momento de la ejecucion:

```hcl
data "aws_secretsmanager_secret_version" "db" {
  secret_id = "rds/produccion"
}

resource "aws_db_instance" "main" {
  username = jsondecode(data.aws_secretsmanager_secret_version.db.secret_string)["username"]
  password = jsondecode(data.aws_secretsmanager_secret_version.db.secret_string)["password"]
}
```

```hcl
data "azurerm_key_vault_secret" "db_password" {
  name         = "db-password"
  key_vault_id = azurerm_key_vault.main.id
}
```

La diferencia fundamental con Vault reside en el modelo de precios (AWS Secrets Manager cobra por llamada), en la rotacion automatica (RDS puede rotar automaticamente secretos en Secrets Manager) y en la integracion con IAM o Azure AD para el control de acceso. En entornos multicloud o hibridos, Vault sigue siendo la opcion mas coherente por su independencia del proveedor.

> El data source de un secreto se evalua durante el plan y el apply. Si el secreto se rota entre ambos pases, el plan puede mostrar un resultado distinto al apply. Para mitigarlo, conviene leer el secreto en un unico paso o utilizar el modo `refresh` en el plan.

## Buenas practicas adicionales

Ningun secreto debe residir en el repositorio. Herramientas como `git-secrets`, `truffleHog` o `Gitleaks` pueden escanear el historial en busca de fugas pasadas. Para entornos locales, un fichero `terraform.tfvars` omitido via `.gitignore` es aceptable como solucion de desarrollo, pero nunca debe contener credenciales de produccion.

La combinacion optima consiste en utilizar Terraform Cloud o un backend de Vault como fuente de secretos, marcar todas las variables y outputs sensibles con `sensitive = true`, y proteger el state remoto para que los valores no queden accesibles en reposo. La gestion de secretos no es un paso aislado, sino una capa transversal que atraviesa cada recurso declarado en la configuracion.

