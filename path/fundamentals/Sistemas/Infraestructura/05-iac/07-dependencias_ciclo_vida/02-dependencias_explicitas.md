# Dependencias explicitas

Existen situaciones donde el grafo de dependencias construido automaticamente por Terraform a partir de referencias directas resulta insuficiente. Cuando un recurso requiere que otro haya sido creado, modificado o destruido antes que el, pero no existe ninguna referencia textual entre sus atributos, el mecanismo de dependencias implicitas no puede inferir la relacion. El meta-argumento `depends_on` permite al operador declarar estas relaciones de forma explotica, forzando a Terraform a respetar un orden de ejecucion que de otro modo no detectaria.

El uso de `depends_on` es una herramienta poderosa pero que debe aplicarse con criterio. Cada declaracion explicita crea una dependencia unidireccional que Terraform trata como un arco adicional en el grafo de ejecucion. Esto significa que el recurso destino siempre esperara a que el recurso origen haya completado su operacion correspondiente, independientemente de si existen o no valores que se transfieran entre ellos. Un uso excesivo o injustificado de `depends_on` puede degradar el paralelismo del despliegue y generar tiempos de ejecucion innecesariamente largos.

## Cuando las dependencias implicitas no alcanzan

El escenario mas comun donde `depends_on` resulta necesario es cuando dos recursos tienen una relacion operativa pero no tecnica. Un servicio que consume una API requiere que la politica IAM associada al rol del servicio haya sido propagada por el proveedor cloud, pero no existe un atributo en el recurso del servicio que referencie la politica. De forma similar, un balanceador de carga interno puede necesitar que un registro DNS interno apunte a su direccion antes de que las pruebas de salud puedan completarse correctamente.

```hcl
resource "aws_iam_role_policy_attachment" "lambda_exec" {
  role       = aws_iam_role.lambda_role.name
  policy_arn = aws_iam_policy.lambda_policy.arn
}

resource "aws_lambda_function" "processor" {
  function_name = "processor"
  runtime       = "python3.12"
  handler       = "main.handler"
  role          = aws_iam_role.lambda_role.arn
  filename      = "lambda.zip"

  depends_on = [aws_iam_role_policy_attachment.lambda_exec]
}
```

Sin `depends_on`, Terraform crearia la funcion Lambda y la politica IAM simultaneamente o en un orden basado unicamente en las referencias directas. La funcion Lambda referencia el rol, pero no la politica adjunta. Si la Lambda se invoca inmediatamente despues de su creacion antes de que la propagacion del permiso se complete, fallaria con un error de acceso denegado. La dependencia explicita garantiza que la politica este adjunta antes de que Terraform considere la creacion de la Lambda como completada.

> Un error clasico es agregar `depends_on` de forma preventiva "por si acaso". Esto oculta problemas reales de diseno y dificulta la depuracion. Solo debe usarse cuando existe una dependencia operativa que Terraform no puede inferir.

## Patrones de uso comunes

Uno de los patrones mas frecuentes es la proteccion de recursos criticos que no deben eliminarse accidentalmente. Aunque `lifecycle.prevent_destroy` es el mecanismo mas directo para este proposito, `depends_on` puede complementarse para garantizar que ciertos recursos de soporte existan antes de eliminar un recurso principal.

Otro patron relevante es la sincronizacion con servicios externos. Cuando se configura un dominio en Route53 y se necesita que la propagacion de los servidores DNS autoritativos se complete antes de ejecutar validaciones SSL con Let's Encrypt, la unica forma de asegurar el orden es mediante una dependencia explicita.

```hcl
resource "aws_route53_zone" "principal" {
  name = "ejemplo.com"
}

resource "aws_route53_record" "validacion" {
  zone_id = aws_route53_zone.principal.zone_id
  name    = "_acme-challenge.ejemplo.com"
  type    = "TXT"
  records = ["validacion-temporal"]
}

resource "tls_certificado" "web" {
  private_key_pem = tls_private_key.web.private_key_pem

  lifecycle {
    create_before_destroy = true
  }

  subject {
    common_name  = "ejemplo.com"
    organization = "Mi Org"
  }

  depends_on = [aws_route53_record.validacion]
}
```

En este ejemplo, la generacion del certificado TLS depende de que el registro DNS de validacion exista y haya sido propagado. Sin la dependencia explicita, Terraform intentaria generar el certificado antes de que el registro DNS este disponible, y la validacion ACME fallaria.

## Dependencias implicitas vs explicitas: criterios de decision

| Criterio | Dependencia implicita | Dependencia explicita |
|---|---|---|
| Deteccion | Automatica por referencias | Manual con `depends_on` |
| Mantenimiento | Se actualiza sola al cambiar referencias | Requiere revision manual |
| Precision | Exacta: solo genera la dependencia necesaria | Puede ser excesiva si se aplica indiscriminadamente |
| Rendimiento | No afecta el paralelismo innecesariamente | Puede reducir el paralelismo |
| Uso recomendado | Siempre que exista una referencia | Solo cuando no hay alternativa |

La regla general es que si un recurso necesita el valor de otro recurso, debe referenciarlo directamente en lugar de usar `depends_on`. La referencia directa no solo establece la dependencia de forma implicita, sino que ademas transfiere el valor real del atributo, lo cual es mas robusto y declarativo. `depends_on` solo debe emplearse cuando la relacion es puramente operativa: un recurso necesita que otro exista, pero no necesita ninguno de sus atributos.

```hcl
resource "aws_db_instance" "principal" {
  engine               = "postgres"
  instance_class       = "db.t3.micro"
  allocated_storage    = 20
  skip_final_snapshot  = true
}

resource "aws_lambda_function" "migrador" {
  function_name = "migrador"
  runtime       = "python3.12"
  handler       = "migrate.handler"
  role          = aws_iam_role.lambda_role.arn
  filename      = "migrador.zip"

  environment {
    variables = {
      DB_HOST = aws_db_instance.principal.address
      DB_NAME = aws_db_instance.principal.db_name
    }
  }

  depends_on = [aws_db_instance.principal]
}
```

En este caso, el `depends_on` es redundante porque la Lambda ya referencia atributos de la base de datos en las variables de entorno. Terraform detecta esas referencias y crea la dependencia implicita automaticamente. Este tipo de `depends_on` innecesario es un indicador de que el recurso deberia refactorizarse para incorporar todas sus dependencias como referencias directas.

