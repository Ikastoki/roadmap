# Esquema flexible

MongoDB no impone un esquema fijo a nivel de coleccion. Cada documento puede tener su propio conjunto de campos, con tipos de datos distintos para el mismo campo y estructuras anidadas arbitrarias. Esta libertad, heredada directamente del modelo de documentos BSON, es el rasgo mas distintivo de MongoDB frente a las bases de datos relacionales y la fuente tanto de su principal ventaja como de sus riesgos mas significativos.

## Ventajas del esquema flexible

La ausencia de esquema rigido proporciona agilidad en varias dimensiones del desarrollo de software.

La primera es la **adaptacion rapida al cambio**. En un entorno donde los requisitos evolucionan continuamente, anyadir un campo a un documento no requiere migraciones, scripts ALTER TABLE ni ventanas de mantenimiento. El equipo de desarrollo puede modificar el modelo de datos simplemente modificando el codigo de la aplicacion; los documentos existentes conviven con los nuevos sin necesidad de transformacion. Esto resulta especialmente valioso en startups, prototipos y proyectos con ciclos de desarrollo cortos.

La segunda ventaja es la **representacion natural de datos heterogeneos**. Cuando los objetos del dominio tienen variabilidad estructural (productos con distintos atributos segun su categoria, eventos de logging con diferentes campos segun el tipo, perfiles de usuario con campos opcionales), el modelo de documentos refleja esa variabilidad sin necesidad de tablas separadas ni columnas nullable.

La tercera es la **reduccion del acoplamiento entre sistemas**. Diferentes servicios pueden leer y escribir en la misma coleccion con modelos parcialmente distintos, sin necesidad de acordar un esquema compartido. Esto es habitual en arquitecturas de microservicios que operan sobre el mismo agregado de datos.

```javascript
// Dos documentos en la misma coleccion con esquemas distintos
db.productos.insertMany([
  { _id: 1, nombre: "Camiseta", talla: "L", color: "rojo" },
  { _id: 2, nombre: "Lavadora", eficiencia: "A+++", peso_kg: 75 }
]);
```

## Riesgos de la ausencia de esquema

La misma flexibilidad que permite iterar rapidamente puede convertirse en una trampa cuando el equipo crece o el sistema madura.

El riesgo principal es la **inconsistencia de datos**. Sin un esquema que valide la presencia y tipo de los campos, es perfectamente posible almacenar documentos donde un campo que deberia ser numerico contiene un string, o donde faltan campos obligatorios. Estas inconsistencias se propagan silenciosamente y emergen en tiempo de ejecucion como errores dificiles de rastrear.

El segundo riesgo es la **complejidad en las consultas**. Cuando los documentos tienen estructuras variables, las consultas deben contemplar todos los casos posibles: comprobar si un campo existe antes de usarlo, manejar tipos distintos, y lidiar con estructuras anidadas que pueden estar presentes o ausentes.

El tercer riesgo es la **deuda tecnica acumulada**. Sin un esquema explicito, la unica documentacion del modelo de datos es el codigo de la aplicacion. Cuando los desarrolladores rotan, el conocimiento sobre que campos existen, que tipos deben tener y que valores son validos se pierde gradualmente.

> El esquema flexible es una herramienta, no una excusa para la ausencia de disciplina. Un equipo maduro adopta esquemas flexibles donde benefician y validacion donde protegen, combinando ambas estrategias segun el contexto.

## Validacion de documentos con $jsonSchema

MongoDB permite definir reglas de validacion a nivel de coleccion mediante el operador `$jsonSchema`, que implementa el borrador 7 de JSON Schema. La validacion se aplica en cada operacion de insercion y actualizacion, y puede configurarse para rechazar documentos invalidos (`validationLevel: "strict"`, `validationAction: "error"`) o simplemente registrar advertencias (`validationAction: "warn"`).

### Definicion de un esquema

El schema se define al crear la coleccion o se anyade posteriormente con `collMod`:

```javascript
db.createCollection("usuarios", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["nombre", "email", "edad"],
      properties: {
        nombre: {
          bsonType: "string",
          description: "nombre completo del usuario"
        },
        email: {
          bsonType: "string",
          pattern: "^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$"
        },
        edad: {
          bsonType: "int",
          minimum: 0,
          maximum: 150
        },
        direccion: {
          bsonType: "object",
          required: ["calle", "ciudad"],
          properties: {
            calle: { bsonType: "string" },
            ciudad: { bsonType: "string" },
            codigo_postal: { bsonType: "string" }
          }
        },
        roles: {
          bsonType: "array",
          items: {
            enum: ["admin", "user", "moderator"]
          },
          uniqueItems: true
        },
        createdAt: {
          bsonType: "date"
        }
      }
    }
  },
  validationAction: "error"
});
```

### Componentes de $jsonSchema

- **bsonType**: tipo BSON esperado para el campo. Soporta todos los tipos nativos de MongoDB.
- **required**: lista de campos obligatorios a nivel del documento o subdocumento actual.
- **properties**: definicion de cada campo, con sus restricciones (tipo, patron, valores permitidos, rangos).
- **additionalProperties**: controla si se permiten campos no definidos en `properties`. Por defecto es `true`; establecerlo a `false` fuerza un esquema estricto.
- **enum**: lista de valores permitidos para un campo.
- **pattern**: expresion regular que debe cumplir el valor del campo.
- **minimum/maximum**: rango numerico.
- **minLength/maxLength**: longitud de strings.
- **items**: esquema aplicable a cada elemento de un array.

### Estrategia de validacion

La validacion con `$jsonSchema` se combina mejor con el esquema flexible cuando se aplica de forma selectiva:

- **Validacion estricta** en campos criticos (identificadores, fechas, valores monetarios, referencias) donde la inconsistencia tiene coste alto.
- **Validacion permisiva** en campos secundarios o de metadatos donde la flexibilidad sigue siendo valiosa.
- **Schemas versionados**: anyadir un campo `schemaVersion` en los documentos permite aplicar distintas reglas de validacion segun la version, facilitando migraciones graduales.

> `$jsonSchema` no puede hacer cumplir relaciones entre documentos (integridad referencial). Para eso existen las restricciones de clave foranea a nivel de aplicacion o operaciones de agregacion con `$lookup`.

