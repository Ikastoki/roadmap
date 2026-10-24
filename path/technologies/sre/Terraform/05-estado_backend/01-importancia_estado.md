# Importancia del estado

El archivo de estado (`terraform.tfstate`) es el mecanismo mediante el cual Terraform mantiene la correspondencia entre la configuracion declarativa escrita en HCL y los objetos reales existentes en los proveedores cloud. Sin este archivo, Terraform carece de memoria: cada ejecucion comenzaria desde cero, incapaz de determinar que recursos ya existen, cuales deben modificarse y cuales deben eliminarse. El estado no es un registro opcional, es la columna vertebral del modelo de operacion de Terraform.

## Mapeo entre codigo y recursos reales

Cuando Terraform ejecuta `apply`, crea una entrada en el estado para cada recurso definido en los archivos `.tf`. Cada entrada contiene un identificador unico que combina el tipo de recurso, su nombre local y un identificador del proveedor (por ejemplo, `aws_instance.web` mapea al ID real `i-0abcd1234efgh5678`). En ejecuciones posteriores, Terraform consulta este mapeo para decidir que acciones tomar:

- Si un recurso existe en el estado pero no en el codigo, Terraform lo destruye.
- Si un recurso existe en el codigo pero no en el estado, Terraform lo crea.
- Si un recurso existe en ambos pero sus atributos difieren, Terraform lo modifica.

Este proceso de reconciliacion es lo que permite el comportamiento declarativo de Terraform: el usuario describe el estado deseado y la herramienta calcula automaticamente las operaciones necesarias para alcanzarlo partiendo del estado actual registrado.

```json
{
  "version": 4,
  "terraform_version": "1.6.0",
  "resources": [
    {
      "module": "root",
      "mode": "managed",
      "type": "aws_instance",
      "name": "web",
      "provider": "provider[\"registry.terraform.io/hashicorp/aws\"]",
      "instances": [
        {
          "schema_version": 1,
          "attributes": {
            "id": "i-0abcd1234efgh5678",
            "ami": "ami-0c55b159cbfafe1f0",
            "instance_type": "t2.micro"
          }
        }
      ]
    }
  ]
}
```

> El estado almacena el identificador real del recurso en el cloud, no el nombre simbolico definido en HCL. Terraform nunca deberia reconstruir este identificador a partir de los atributos; confia ciegamente en el mapeo almacenado.

## Metadatos y dependencias

El estado no se limita a almacenar atributos. Tambien registra metadatos esenciales para el funcionamiento interno de Terraform:

- **Dependencias**: Terraform construye un grafico de dependencias a partir de las referencias entre recursos en el codigo. Por ejemplo, si `aws_instance.web` referencia `aws_subnet.main.id`, Terraform sabe que la subred debe crearse antes que la instancia. El estado almacena estas dependencias internas para respetar el orden de operaciones.
- **Schema version**: Cada tipo de recurso tiene una version de esquema. Cuando un proveedor actualiza su schema, Terraform migra automaticamente el estado antiguo al nuevo formato durante `terraform apply`.
- **Identidad del proveedor**: El estado registra que proveedor y que version gestionan cada recurso, permitiendo a Terraform invocar el plugin correcto al leer o modificar el recurso.
- **Atributos sensibles**: Los valores marcados como `sensitive` se almacenan en el estado aunque se oculten en la salida de `plan` y `apply`. Por esta razon, el archivo de estado debe tratarse como informacion potencialmente sensible.

```json
{
  "dependencies": [
    "aws_subnet.main",
    "aws_security_group.web_sg"
  ]
}
```

La precision de estas dependencias determina la capacidad de Terraform para paralelizar operaciones. Recursos sin dependencias entre si pueden crearse, modificarse o destruirse concurrentemente, acelerando las ejecuciones.

## Consecuencias de un estado corrupto o desincronizado

Un estado que no refleja la realidad tiene consecuencias graves:

- **Deriva no detectada**: Si alguien modifica un recurso manualmente en la consola cloud, Terraform no lo sabra hasta el proximo `plan`. Si el estado no se actualiza, Terraform sobrescribira el cambio manual en el proximo `apply`, potencialmente causando perdida de configuracion.
- **Duplicacion de recursos**: Si el estado se pierde, `apply` intentara crear recursos que ya existen, produciendo errores por nombre duplicado o, peor aun, creando recursos paralelos no deseados.
- **Destruccion accidental**: Si se elimina un recurso del estado (con `state rm`) sin eliminar el recurso real, Terraform perdera el rastro. Un `destroy` posterior no lo tocara, quedando como un recurso huerfano que sigue generando costos.
- **Conflictos en equipo**: Cuando dos personas ejecutan `apply` simultaneamente sin un mecanismo de bloqueo, el ultimo en escribir sobrescribe el estado del primero, provocando perdida de cambios o corrupcion.

## El estado como fuente de verdad operativa

En la practica, el estado funciona como la unica fuente de verdad operativa para Terraform. El codigo HCL describe el *deseo*, el estado describe la *realidad*, y Terraform actua como el mecanismo de reconciliacion entre ambos. Esta arquitectura de tres capas (codigo, estado, recursos reales) es lo que diferencia a Terraform de herramientas puramente imperativas como scripts de shell o CloudFormation, donde no existe una nocion explicita de estado intermedio.

Mantener el estado intacto, versionado y accesible es tan critico como mantener el propio codigo de infraestructura. Perder el estado no es un inconveniente menor: es un incidente operativo que requiere reconstruccion manual del mapeo entre todos los recursos gestionados.

