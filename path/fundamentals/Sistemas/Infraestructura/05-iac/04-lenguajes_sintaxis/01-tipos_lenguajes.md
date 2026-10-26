# Tipos de lenguajes de definicion

La eleccion del lenguaje en el que se expresa la infraestructura determina no solo la sintaxis concreta, sino el modelo mental que adopta el equipo: declarativo frente a imperativo, dominio especifico frente a general, estatico frente a dinamico. Cada herramienta de IaC ha hecho una apuesta distinta en este espectro y, comprender las razones detras de cada decision, permite elegir con criterio la mas adecuada para un contexto dado.

Un lenguaje de dominio especifico (DSL) esta diseñado para describir infraestructura de forma directa: su sintaxis refleja conceptos de nube, red o almacenamiento sin necesidad de abstracciones generales. Terraform utiliza HCL (HashiCorp Configuration Language), un formato legible tanto por humanos como por maquinas, cuya gramatica se orienta a bloques de recurso, variables y expresiones simples. CloudFormation, por su parte, emplea YAML o JSON: formatos de estructura de datos que, aunque no son lenguajes de programacion en sentido estricto, permiten describir recursos como documentos serializados con un esquema definido por la propia plataforma de AWS.

Los lenguajes de proposito general permiten utilizar la logica de programacion existente en el ecosistema: tipos, funciones, herencia, pruebas unitarias, paquetes y control de versiones. Pulumi acepta TypeScript, Python, Go y C#, lo que abre la posibilidad de reutilizar bibliotecas, aplicar principios SOLID y ejecutar pruebas automaticas sobre la definicion de infraestructura. AWS CDK sigue una filosofia similar pero se centra exclusivamente en el ecosistema AWS, generando plantillas de CloudFormation a partir de codigo en TypeScript, Python, Go o Java.

## Comparativa visual: el mismo recurso en tres lenguajes

Para entender las diferencias practicas, supongamos que se desea crear un bucket de S3 con un nombre predefinido y una etiqueta de entorno. A continuacion se muestra la misma definicion en HCL, TypeScript (Pulumi) y YAML (CloudFormation).

**Terraform (HCL)**

```hcl
resource "aws_s3_bucket" "datos" {
  bucket = "mi-bucket-datos"
  tags = {
    Environment = var.entorno
  }
}
```

HCL organiza la configuracion en bloques identificados por un tipo (`resource`), un proveedor (`aws_s3_bucket`) y un nombre local (`datos`). Las variables se referencian con el prefijo `var.` y las expresiones se evaluan en un lenguaje propio de Terraform, limitado pero predecible.

**Pulumi (TypeScript)**

```typescript
import * as aws from "@pulumi/aws";

const bucket = new aws.s3.Bucket("datos", {
  bucket: "mi-bucket-datos",
  tags: {
    Environment: config.require("entorno"),
  },
});
```

En Pulumi, la infraestructura se expresa como codigo TypeScript convencional. Se crean objetos de tipo `Bucket` mediante constructores que reciben un nombre logico y un objeto con las propiedades. La logica del lenguaje host (condicionales, bucles, funciones) esta disponible de forma natural.

**CloudFormation (YAML)**

```yaml
Resources:
  DatosBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: mi-bucket-datos
      Tags:
        - Key: Environment
          Value: !Ref Entorno
```

CloudFormation utiliza la estructura jerarquica de YAML para anidar propiedades. Las referencias a parametros y otros recursos se realizan con funciones intrinsecas como `!Ref`, `!Sub` o `!GetAtt`. No existe un motor de ejecucion independiente: la propia plataforma de AWS interpreta el documento.

## Tabla comparativa

| Aspecto | HCL (Terraform) | YAML/JSON (CloudFormation) | TypeScript/Python (Pulumi) |
|---|---|---|---|
| Tipo de lenguaje | DSL declarativo | Formato de datos serializado | General imperativo |
| Curva de aprendizaje | Baja para ops, media para devs | Baja (solo estructura JSON) | Baja para devs, media para ops |
| Expresividad logica | Limitada (condicionales simples) | Funciones intrinsecas limitadas | Completa (bucles, funciones, clases) |
| Pruebas unitarias | Herramientas externas | No disponibles nativamente | Integradas en el lenguaje |
| Reutilizacion de codigo | Modulos Terraform | Nested stacks, macro | Paquetes npm/pip, clases |
| Validacion de esquema | `terraform validate` | API de CloudFormation | Compilador del lenguaje host |
| Renderizado legible | Alto | Alto (YAML), bajo (JSON) | Depende del estilo del autor |

> HCL fue disenado como un lenguaje declarativo deliberadamente limitado. Su intencion es que la infraestructura se exprese de forma estatica, sin efectos secundarios. Las herramientas como Pulumi o CDK rompen esa barrera al introducir un lenguaje completo, lo cual aporta flexibilidad pero tambien el riesgo de crear infraestructura con comportamiento no determinista.

La eleccion entre un DSL y un lenguaje de proposito general no es estrictamente tecnica: depende del perfil del equipo, la complejidad de la infraestructura y la necesidad de integrar la definicion de infraestructura con flujos de trabajo de desarrollo existentes. Los DSLs son ideales para equipos de operaciones que priorizan legibilidad y simplicidad. Los lenguajes generales encajan mejor cuando la infraestructura se versiona junto al codigo de la aplicacion y se somete a los mismos procesos de revision y prueba.

