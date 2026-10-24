# Indices compuestos

Un indice compuesto indexa dos o mas campos en una unica estructura B-Tree. La clave del indice es la concatenacion de los valores de los campos en el orden definido. Esto permite que una sola estructura sirva a multiples patrones de consulta distintos, siempre que respeten el orden de prefijos.

## Orden de campos

En un indice compuesto `{ a: 1, b: -1, c: 1 }`, los documentos se ordenan primero por `a` ascendente, dentro de cada valor de `a` por `b` descendente, y dentro de cada par (a, b) por `c` ascendente.

El orden de los campos en la definicion determina que consultas puede servir el indice. No es lo mismo `{ a: 1, b: 1 }` que `{ b: 1, a: 1 }`.

```javascript
db.ventas.createIndex({ fecha: -1, total: 1 })
```

Este indice ordena primero por `fecha` descendente, y dentro de cada fecha por `total` ascendente. Servira eficientemente consultas que filtren por `fecha` y ordenen por `total`, pero no consultas que filtren solo por `total`.

## Prefix rule (regla de prefijo)

La regla de prefijo establece que un indice compuesto puede utilizarse para consultas que involucren un prefijo (desde la izquierda) de sus campos, pero no para consultas que salten campos del principio.

Dado un indice `{ a: 1, b: 1, c: 1 }`:

| Consulta | Usa indice |
|----------|-----------|
| `{ a: 1 }` | Si (prefijo de 1 campo) |
| `{ a: 1, b: 1 }` | Si (prefijo de 2 campos) |
| `{ a: 1, b: 1, c: 1 }` | Si (prefijo completo) |
| `{ b: 1 }` | No (salta `a`) |
| `{ c: 1 }` | No (salta `a` y `b`) |
| `{ a: 1, c: 1 }` | Parcial: usa `a` para filtrar pero no `c` |

La implicacion practica es que el campo mas restrictivo o mas usado en igualdad debe ir primero. Un error comun es poner primero el campo con mas cardinalidad pensando que asi se filtra mas rapido, cuando lo que importa es cuantos documentos descarta cada nivel.

```javascript
// Dado un indice { ciudad: 1, edad: 1 }
db.usuarios.find({ ciudad: "Madrid", edad: { $gt: 18 } })
// Usa el indice completo: primero filtra por ciudad, luego por edad

// Dado un indice { edad: 1, ciudad: 1 }
db.usuarios.find({ ciudad: "Madrid", edad: { $gt: 18 } })
// Usa solo edad: el indice no puede saltar a ciudad sin fijar primero edad
```

> La regla de prefijo tambien se aplica a la ordenacion. Un indice `{ a: 1, b: -1 }` puede ordenar por `{ a: 1 }` ascendente o por `{ a: 1, b: -1 }`, pero no por `{ b: -1 }` solo.

## ESR Rule (Equality, Sort, Range)

La ESR Rule es la heuristica fundamental para disenar indices compuestos. Establece el orden optimo de los campos:

1. **Equality**: primero los campos que se filtran con igualdad (`$eq`, `$in`). Estos son los que mas reducen el espacio de busqueda.
2. **Sort**: despues los campos utilizados en `sort()`. Colocarlos aqui evita una ordenacion en memoria (bloqueante para la consulta).
3. **Range**: al final los campos con filtros de rango (`$gt`, `$lt`, `$gte`, `$lte`, `$ne`).

```javascript
// Consulta: filtrar por ciudad y activo, ordenar por fecha
db.usuarios.find(
  { ciudad: "Madrid", activo: true },
  { fecha: -1 }
)

// Indice optimo segun ESR:
// Equality: activo (o ciudad), Sort: fecha, Range: (ninguno aqui)
db.usuarios.createIndex({ activo: 1, ciudad: 1, fecha: -1 })
// O: { ciudad: 1, activo: 1, fecha: -1 }
```

El motivo es que MongoDB puede recorrer el indice en orden directamente si los campos de sort son consecutivos a los de equality. Si el campo de sort esta despues de un range, MongoDB debe ordenar en memoria, lo cual es mas costoso.

```javascript
// ESR aplicado con range
// Consulta: ciudad exacta, edad > 18, ordenado por fecha
db.usuarios.find(
  { ciudad: "Madrid", edad: { $gt: 18 } }
).sort({ fecha: -1 })

// Indice: Equality (ciudad), Sort (fecha), Range (edad)
db.usuarios.createIndex({ ciudad: 1, fecha: -1, edad: 1 })
```

> La ESR Rule no es una ley matematica sino una guia. Siempre hay que verificar con `explain()` si el planificador esta usando el indice como se espera. En algunos casos con rangos muy selectivos puede convenir alterar el orden.

## Covered query con indices compuestos

Un indice compuesto puede servir consultas cubiertas siempre que todos los campos requeridos por el filtro y la proyeccion esten contenidos en el indice.

```javascript
db.usuarios.createIndex({ email: 1, nombre: 1, rol: 1 })

// Covered query
db.usuarios.find(
  { email: "user@example.com" },
  { _id: 0, email: 1, nombre: 1, rol: 1 }
)
// totalDocsExamined: 0
```

Para que una consulta sea cubierta con un indice compuesto:

1. Todos los campos del filtro deben estar en el indice.
2. Todos los campos de la proyeccion deben estar en el indice.
3. `_id` debe excluirse explicitamente en la proyeccion si no forma parte del indice.
4. Ningun campo del indice puede ser un array (no puede ser multikey).

Cuando se cumple, la consulta no necesita acceder a los documentos, lo que la hace extremadamente rapida. La salida de `explain("executionStats")` mostrara `totalDocsExamined: 0` e `IXSCAN` como unica etapa, sin `FETCH`.

```javascript
// Si el indice es { ciudad: 1, nombre: 1, _id: 1 } y el _id esta incluido
db.usuarios.find(
  { ciudad: "Madrid" },
  { _id: 1, ciudad: 1, nombre: 1 }
)
// Covered query porque _id esta en el indice
```

> En aplicaciones con alta tasa de lecturas, disenar indices compuestos que cubran las consultas mas frecuentes puede reducir drasticamente la presion sobre el cache de WiredTiger, ya que los datos se obtienen exclusivamente del indice, que suele ser mucho mas pequeno que la coleccion completa.

