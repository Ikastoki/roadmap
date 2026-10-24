# BSON y ObjectId

MongoDB no almacena los documentos en JSON puro, sino en una representacion binaria denominada BSON (Binary JSON). BSON extiende JSON con tipos de datos que no existen en el estandar original, como fechas, numeros de precision arbitraria y datos binarios. Esta representacion no solo amplia el espectro semantico de los documentos, sino que optimiza la eficiencia de escaneo y serializacion: BSON esta disenado para ser recorrido rapidamente, algo critico en un motor de base de datos que debe comparar y proyectar campos continuamente.

## Estructura de BSON

BSON es un formato serializado que representa documentos como una secuencia de elementos ordenados, donde cada elemento consta de un tipo (1 byte), un nombre del campo (string terminado en nulo) y un valor. La cabecera del documento contiene la longitud total en bytes, lo que permite saltar documentos completos sin parsear su interior. Esta caracteristica es fundamental para el rendimiento del motor de agregacion y los indices.

Cada documento BSON tiene un limite de 16 MB, disenado para evitar fragmentacion excesiva de la memoria y garantizar que las operaciones de red sean atomicas a nivel de documento.

## Tipos de datos en BSON

BSON incorpora una decena de tipos que van mucho mas alla de los seis tipos basicos de JSON (string, number, boolean, null, array, object):

- **String**: secuencia de caracteres UTF-8. En BSON, los strings almacenan explicitamente su longitud, lo que permite incluir caracteres nulos sin riesgo de truncamiento.
- **Int32**: entero con signo de 32 bits (4 bytes). Es el tipo numerico mas eficiente y el que se usa por defecto para valores enteros en la mayoria de los drivers.
- **Int64**: entero con signo de 64 bits (8 bytes). Necesario para valores que exceden el rango de Int32 (~2.1 mil millones).
- **Double**: numero en coma flotante de 64 bits segun IEEE 754. Es el tipo por defecto para numeros con decimales en MongoDB.
- **Decimal128**: numero decimal de 128 bits con precision exacta, implementado segun la especificacion IEEE 754-2008. Indispensable para aplicaciones financieras donde los errores de redondeo de coma flotante son inaceptables.
- **Boolean**: valor `true` o `false`, almacenado en un unico byte.
- **Null**: ausencia intencionada de valor. Diferencia entre un campo que no existe y un campo que existe pero no tiene valor.
- **ObjectId**: identificador unico de 12 bytes, generado automaticamente como valor por defecto para `_id`.
- **Date**: instante de tiempo expresado como milisegundos desde la epoch Unix (1 de enero de 1970), almacenado como Int64 con signo. Soporta fechas desde el anyo 1970 hasta aproximadamente el 3000.
- **Array**: lista ordenada de valores, representada internamente como un objeto con claves numericas consecutivas ("0", "1", "2"...).
- **Object**: documento embebido (subdocumento), que a su vez es un BSON completo.
- **Binary**: secuencia de bytes arbitrarios, con un subtipo que indica la naturaleza de los datos (generico, UUID, MD5, etc.). Util para almacenar imagenes, archivos o datos cifrados.
- **Regex**: expresion regular compilada, almacenable directamente en el documento para su uso en consultas.

> Elegir el tipo numerico adecuado tiene impacto directo en el rendimiento y el espacio en disco. Usar Double para valores monetarios puede introducir errores de redondeo; Decimal128 es la opcion correcta en esos casos, aunque ocupa el doble de espacio.

## ObjectId

El campo `_id` es el identificador unico de cada documento en una coleccion de MongoDB. Si no se especifica un valor para `_id` al insertar un documento, MongoDB genera automaticamente un ObjectId.

### Estructura interna

Un ObjectId ocupa exactamente 12 bytes, organizados en cuatro componentes:

```
|-- 4 bytes --|-- 3 bytes --|-- 2 bytes --|-- 3 bytes --|
  timestamp     machine id    process id    counter
```

1. **Timestamp** (4 bytes): numero de segundos desde la epoch Unix. Proporciona orden cronologico aproximado y permite ordenar ObjectIds por momento de creacion sin necesidad de un campo de fecha adicional.
2. **Identificador de maquina** (3 bytes): hash del nombre del host o direccion MAC del servidor donde se genero el ObjectId. Garantiza unicidad entre maquinas distintas.
3. **Identificador de proceso** (2 bytes): PID del proceso que genera el ObjectId. Distingue ObjectIds generados por distintos procesos en la misma maquina.
4. **Contador** (3 bytes): valor incremental que empieza en un valor aleatorio y se incrementa en cada generacion. Asegura unicidad incluso para ObjectIds generados en el mismo proceso y segundo.

Esta estructura asegura que ObjectIds sean unicos globalmente sin necesidad de coordinacion centralizada, lo que permite a los drivers generarlos localmente sin consultar al servidor.

### Propiedades del ObjectId

- **Unico globalmente**: la combinacion de los cuatro componentes garantiza unicidad incluso en entornos distribuidos con multiples servidores y procesos.
- **Orden cronologico**: el timestamp inicial permite ordenar ObjectIds por tiempo de creacion. Sin embargo, el orden no es estrictamente secuencial porque el contador se reinicia cada segundo.
- **Monotono creciente**: dentro del mismo segundo y proceso, los ObjectIds son estrictamente crecientes debido al contador.
- **Compacto**: 12 bytes (24 caracteres hexadecimales en representacion string) es significativamente menor que un UUID de 16 bytes o 36 caracteres.

> No se debe confiar en ObjectId para ordenar documentos por tiempo de creacion con precision inferior a un segundo. Si se necesita orden exacto, es preferible anyadir un campo timestamp explicito.

### Metodos de generacion alternativa

Para el campo `_id` se puede usar cualquier tipo de dato, siempre que sea unico dentro de la coleccion. Es comun usar:
- **UUID estandar**: secuencia de 16 bytes, adecuada cuando se requiere interoperabilidad con sistemas externos.
- **Entero secuencial**: util en migraciones desde SQL o cuando se necesita un identificador legible.
- **String personalizado**: combinacion de prefijos semanticos y contadores.
- **Compuesto**: objeto con varios campos que en conjunto forman una clave unica.

La eleccion del tipo de `_id` afecta al rendimiento de las inserciones: los `_id` monotonicos (ObjectId, entero secuencial) favorecen el rendimiento de escritura porque los indices se mantienen compactos, mientras que los `_id` aleatorios (UUID v4) provocan fragmentacion del arbol B+ del indice primario.

