# Protocol Buffers

Protocol Buffers (protobuf) es un mecanismo de serializacion binaria desarrollado por Google para el intercambio eficiente de datos estructurados. A diferencia de JSON y XML, que son formatos de texto, protobuf serializa los datos en un formato binario compacto que reduce drasticamente el tamano de los mensajes y acelera los tiempos de serializacion y deserializacion. Es el formato subyacente en gRPC.

## Esquema .proto

Los datos se definen en un archivo de esquema con extension `.proto`. Cada mensaje es una estructura tipada con campos numerados.

```protobuf
syntax = "proto3";

package com.ejemplo;

option java_multiple_files = true;
option java_package = "com.ejemplo.protobuf";

message Usuario {
  int64 id = 1;
  string nombre = 2;
  string email = 3;
  repeated string roles = 4;
}
```

Cada campo tiene:
- Un **tipo** (int32, int64, float, double, bool, string, bytes, enum, otros mensajes).
- Un **nombre** unico dentro del mensaje.
- Un **numero** unico que identifica el campo en la codificacion binaria. No debe cambiarse nunca una vez en uso, ya que los datos serializados usan este numero, no el nombre.

## Compilacion del esquema

El archivo `.proto` se compila con el compilador `protoc` para generar clases Java:

```bash
protoc --java_out=src/main/java usuario.proto
```

Esto genera la clase `Usuario` con builders, getters, setters y metodos de serializacion.

## Uso en Java

```java
import com.ejemplo.protobuf.Usuario;
import com.ejemplo.protobuf.UsuarioOuterClass;

// Construir mensaje
Usuario usuario = Usuario.newBuilder()
        .setId(1L)
        .setNombre("Ana Garcia")
        .setEmail("ana@example.com")
        .addRoles("admin")
        .addRoles("usuario")
        .build();

// Serializar a binario
byte[] datos = usuario.toByteArray();

// Deserializar
Usuario otro = Usuario.parseFrom(datos);
System.out.println(otro.getNombre()); // Ana Garcia
```

## Ventajas frente a JSON/XML

### Tamano

Un mensaje protobuf tipico es 3-10 veces mas pequeno que su equivalente en JSON y 10-20 veces mas pequeno que XML. Esto se debe a:

- Formato binario sin etiquetas de texto repetitivas.
- Codificacion de enteros con varint (los numeros pequenos usan menos bytes).
- Tipos fijos para datos de tamano constante.

### Velocidad

La serializacion y deserializacion son significativamente mas rapidas que JSON (2-5x) y XML (10-50x) porque no hay parsing de texto ni construccion de arboles DOM.

### Tipado estricto

El esquema `.proto` define tipos exactos. No hay ambiguedad entre "123" como string vs 123 como int. El compilador genera codigo con tipos Java concretos, eliminando errores de casting.

### Compatibilidad hacia atras

Los campos numerados permiten anadir nuevos campos sin romper mensajes antiguos. Los clientes viejos ignoran los campos que no conocen.

```protobuf
message Usuario {
  int64 id = 1;
  string nombre = 2;
  string email = 3;
  // Campo anadido en version 2
  string telefono = 5;
}
```

> Las reglas de evolucion de protobuf son estrictas: un campo numerado no debe reutilizarse ni cambiarse de tipo. Los campos eliminados deben marcarse como `reserved` para evitar su reutilizacion accidental.

## Desventajas

- El esquema debe compilarse previamente; no se puede construir un mensaje arbitrario en tiempo de ejecucion como con JSON.
- Los datos no son legibles por humanos sin herramientas especificas.
- La depuracion requiere conversion a JSON o uso de `protoc --decode`.
- No es adecuado para APIs publicas donde la simplicidad del consumo supera a la eficiencia.

## Dependencia Maven

```xml
<dependency>
    <groupId>com.google.protobuf</groupId>
    <artifactId>protobuf-java</artifactId>
    <version>4.28.0</version>
</dependency>
```

## Comparativa

| Caracteristica | JSON | XML | Protobuf |
|---|---|---|---|
| Formato | Texto | Texto | Binario |
| Legible | Si | Si | No |
| Tamano | Medio | Grande | Pequeno |
| Velocidad | Media | Lenta | Rapida |
| Tipado | Debil | XSD | Fuerte |
| Versionado | Manual | XSD | Nativo |
| Streaming | Parcial | Si | Parcial |
| Herramientas | Universales | Maduras | Requiere compilacion |

> Protocol Buffers es la mejor opcion para comunicacion interna entre microservicios donde el rendimiento y el ancho de banda importan. Para APIs publicas expuestas a clientes externos, JSON sigue siendo el estandar por su legibilidad y ubicuidad.
