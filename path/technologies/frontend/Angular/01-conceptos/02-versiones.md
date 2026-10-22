<h1 align="center">Versiones</h1>

<h2>📑 Contenido</h2>

- [Un ciclo de versiones predecible](#un-ciclo-de-versiones-predecible)
- [Versiones clave y lo que necesitas saber](#versiones-clave-y-lo-que-necesitas-saber)
- [¿Cómo saber qué versión estás usando?](#cómo-saber-qué-versión-estás-usando)
- [¿Cómo actualizar Angular?](#cómo-actualizar-angular)

## Un ciclo de versiones predecible

Una de las ventajas de Angular es su ciclo de lanzamiento bien definido y predecible. Esto permite a los desarrolladores planificar actualizaciones y mantenerse al día con las últimas funcionalidades.

- Versiones mayores: Se lanzan cada seis meses. Estas pueden incluir cambios importantes que a veces requieren una adaptación por parte del desarrollador para migrar el código.

- Versiones menores: Se lanzan una o tres veces por cada versión mayor. Son completamente compatibles con versiones anteriores, lo que significa que no rompen tu código existente.

- Versiones de parche: Se publican cada semana para corregir errores menores y vulnerabilidades.

## Versiones clave y lo que necesitas saber

Aunque Angular empezó como AngularJS (una versión completamente diferente), a partir de Angular 2, el framework se ha mantenido en una evolución continua.

- **Angular 2**: Fue una reescritura completa del framework original (AngularJS). Introdujo los conceptos que aún usamos hoy: componentes, TypeScript y la línea de comandos (`Angular CLI`).

- **Angular 4**: Introdujo mejoras de rendimiento y redujo el tamaño del código generado.

- **Angular 6**: Incluyó la integración con el árbol de módulos de la versión 5 de RxJS y la introducción de `ng add` y `ng update` en la CLI para facilitar la gestión de dependencias.

- **Angular 9**: El compilador Ivy se activó por defecto. Ivy es un compilador de nueva generación que hace que el código sea más pequeño, más rápido y más fácil de depurar. Este fue un cambio muy importante.

- **Angular 14**: Introdujo los componentes independientes (`standalone components`) como una característica estable, lo que permite crear componentes, directivas y pipes sin tener que declararlos en un `NgModule`. Este es un cambio fundamental y la forma preferida de desarrollo moderno.

- **Angular 16**: Introdujo las señales (`signals`) como una nueva forma de manejar la reactividad en la aplicación, mejorando aún más el rendimiento y simplificando la gestión del estado.

## ¿Cómo saber qué versión estás usando?

Para verificar la versión de Angular y de la CLI que tienes instalada en tu proyecto, puedes usar el siguiente comando en tu terminal, dentro del directorio del proyecto:

```bash
ng version
```

Esto te mostrará una tabla con la versión de Angular CLI, de Node.js y de los paquetes principales de Angular.

## ¿Cómo actualizar Angular?

Angular CLI tiene un comando muy útil para ayudarte a actualizar tu proyecto a una versión más reciente de forma segura.

```bash
ng update
```

Este comando analiza tu proyecto, te recomienda las actualizaciones disponibles y las ejecuta, haciendo los cambios necesarios de forma automática.
