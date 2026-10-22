<h1 align='center'>Módulos</h1>

<h2>📑 Contenido</h2>

- [Módulos](#módulos)
- [¿Qué son los módulos?](#qué-son-los-módulos)
  - [Los módulos son utilizados para:](#los-módulos-son-utilizados-para)
  - [Características clave de los módulos en JavaScript:](#características-clave-de-los-módulos-en-javascript)
- [Programas modulares](#programas-modulares)
  - [Ventajas de los programas modulares](#ventajas-de-los-programas-modulares)
- [Uso de módulos en JavaScript](#uso-de-módulos-en-javascript)
  - [Exportación de módulos](#exportación-de-módulos)
    - [Exportación nombrada (Named Export)](#exportación-nombrada-named-export)
    - [Exportación por defecto (Default Export)](#exportación-por-defecto-default-export)
  - [Importación de módulos](#importación-de-módulos)
    - [Importación de exportaciones nombradas](#importación-de-exportaciones-nombradas)
    - [Importación de la exportación por defecto](#importación-de-la-exportación-por-defecto)

## Módulos

Un módulo es una parte independiente de código que tiene la capacidad de encapsular funcionalidad y compartirla con otros módulos o archivos del mismo proyecto. En la mayoría de los lenguajes de programación modernos, los módulos juegan un papel fundamental para organizar y estructurar aplicaciones grandes y complejas. JavaScript no es una excepción, y aunque no siempre tuvo un sistema nativo de módulos, a partir de la especificación ECMAScript 6 (ES6), se introdujo el concepto de módulos en el lenguaje.

En este capítulo, veremos qué son los módulos en JavaScript, cómo podemos trabajar con ellos y por qué son tan importantes para la construcción de programas modulares.

## ¿Qué son los módulos?

Un módulo es esencialmente un archivo de código que contiene definiciones de funciones, objetos, clases, o cualquier otra construcción de código que se puede exportar e importar en otros archivos. En lugar de tener una gran cantidad de código en un solo archivo que puede volverse difícil de mantener, los módulos permiten dividir el código en unidades más pequeñas, lógicas y reutilizables.

### Los módulos son utilizados para:

**Organización:** Permiten organizar mejor el código dividiéndolo en partes lógicas más pequeñas.
**Reutilización:** Una vez que un módulo está creado, puede ser reutilizado en diferentes partes de la aplicación o incluso en otros proyectos.
**Encapsulación:** Los módulos pueden ocultar detalles de implementación, exponiendo solo lo necesario al exterior.
**Mantenibilidad:** Al tener el código separado en distintos módulos, es más fácil mantener y modificar una parte sin afectar otras.

### Características clave de los módulos en JavaScript:

- **Independencia:** Cada módulo tiene su propio contexto, por lo que las variables y funciones definidas dentro de un módulo no interfieren con las de otros módulos (a menos que se exporten explícitamente).
- **Exportar e importar:** Los módulos permiten la exportación de partes de su código (funciones, objetos, variables) que pueden ser luego importadas en otros módulos.
- **Carga diferida:** Los módulos pueden ser cargados de forma perezosa (lazy loading), es decir, solo cuando son necesarios.

## Programas modulares

Un programa modular es una aplicación construida dividiendo el código en múltiples módulos. En lugar de escribir todo el código en un solo archivo enorme (que sería difícil de gestionar), un enfoque modular divide la funcionalidad en componentes más pequeños, cada uno con su propio propósito específico.

### Ventajas de los programas modulares

- **Escalabilidad:** En aplicaciones grandes, el código puede crecer mucho. Dividirlo en módulos hace que sea más fácil de escalar, ya que se pueden agregar nuevas funcionalidades como módulos independientes.
- **Reutilización:** Un módulo escrito para una parte de una aplicación puede ser reutilizado en otra parte, o incluso en proyectos diferentes, sin tener que duplicar código.
- **Mantenimiento:** Al modificar una funcionalidad, es probable que sólo necesites modificar un módulo específico, en lugar de buscar en un archivo monolítico. Esto reduce la probabilidad de errores y facilita el mantenimiento a largo plazo.
- **Colaboración:** En un equipo de desarrollo, diferentes desarrolladores pueden trabajar en diferentes módulos de manera simultánea sin interferir en el trabajo de otros.
- **Pruebas:** Es más sencillo probar módulos individuales que una aplicación completa. Cada módulo puede tener sus propios tests unitarios.

## Uso de módulos en JavaScript

JavaScript implementa módulos de forma nativa a través de la sintaxis de exportación e importación.

### Exportación de módulos

Para que otros módulos puedan utilizar partes de un módulo, esas partes deben ser exportadas. Existen dos tipos de exportaciones en JavaScript:

#### Exportación nombrada (Named Export)

Puedes exportar múltiples elementos (funciones, variables, clases) desde un módulo utilizando un nombre específico para cada uno.

```js
// archivo: mathModule.js
export function sumar(a, b) {
  return a + b;
}

export const PI = 3.14159;
```

#### Exportación por defecto (Default Export)

En este caso, un módulo puede exportar un único valor predeterminado. No es necesario utilizar un nombre cuando se importa el valor por defecto.

```js
// archivo: greetingModule.js
export default function saludar(nombre) {
  return `Hola, ${nombre}`;
}
```

### Importación de módulos

Una vez que un módulo ha exportado sus elementos, otro archivo puede importarlos. Dependiendo del tipo de exportación, la sintaxis para importar varía.

#### Importación de exportaciones nombradas

Se deben importar utilizando los mismos nombres que fueron exportados.

```js
// archivo: main.js
import { sumar, PI } from "./mathModule.js";

console.log(sumar(5, 10)); // 15
console.log(PI); // 3.14159
```

#### Importación de la exportación por defecto

No es necesario usar el mismo nombre al importar un valor por defecto.

```js
// archivo: main.js
import saludar from "./greetingModule.js";

console.log(saludar("Juan")); // "Hola, Juan"
```

Aquí, el nombre de la función saludar podría haber sido cualquier otro, ya que estamos importando la exportación predeterminada.

```

```
