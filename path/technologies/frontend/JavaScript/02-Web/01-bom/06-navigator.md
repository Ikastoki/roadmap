<h1 align='center'>Navigator - Información del Navegador</h1>

<h2>📑 Contenido</h2>

- [Navigator](#navigator)
- [¿Qué es el Objeto navigator?](#qué-es-el-objeto-navigator)
- [Propiedades Principales del Objeto navigator](#propiedades-principales-del-objeto-navigator)
  - [navigator.appName](#navigatorappname)
  - [navigator.appVersion](#navigatorappversion)
  - [navigator.userAgent](#navigatoruseragent)
  - [navigator.platform](#navigatorplatform)
  - [navigator.language](#navigatorlanguage)
  - [navigator.cookieEnabled](#navigatorcookieenabled)
- [Ejemplos de Uso del Objeto navigator](#ejemplos-de-uso-del-objeto-navigator)
  - [Obtener el Nombre del Navegador](#obtener-el-nombre-del-navegador)
  - [Obtener el User Agent](#obtener-el-user-agent)
  - [Verificar si las Cookies Están Habilitadas](#verificar-si-las-cookies-están-habilitadas)
  - [Obtener el Idioma Preferido](#obtener-el-idioma-preferido)
- [Consideraciones](#consideraciones)

## Navigator

El objeto **`navigator`** en JavaScript es parte del **Browser Object Model (BOM)** y proporciona información sobre el navegador web que el usuario está utilizando. Este objeto incluye detalles sobre la versión del navegador, el sistema operativo, la configuración de la conexión y más. La información proporcionada por el objeto `navigator` puede ser útil para adaptar la experiencia del usuario según sus capacidades del navegador.

## ¿Qué es el Objeto navigator?

El objeto **`navigator`** proporciona información sobre el entorno del navegador actual. Esto incluye propiedades que indican el tipo de navegador, su versión, y si el navegador es compatible con ciertas características. A través del objeto `navigator`, los desarrolladores pueden tomar decisiones basadas en la configuración y características del navegador del usuario.

## Propiedades Principales del Objeto navigator

### navigator.appName

Devuelve el nombre del navegador. Sin embargo, esta propiedad no siempre es confiable, ya que puede variar entre diferentes navegadores.

```javascript
console.log(navigator.appName); // Muestra el nombre del navegador
```

### navigator.appVersion

Devuelve la versión del navegador. Esta propiedad incluye información detallada que puede ser útil para el diagnóstico.

```javascript
console.log(navigator.appVersion); // Muestra la versión del navegador
```

### navigator.userAgent

Devuelve el string del agente de usuario (user agent), que contiene información sobre el navegador y el sistema operativo.

```javascript
console.log(navigator.userAgent); // Muestra el user agent
```

### navigator.platform

Devuelve el nombre del sistema operativo en el que se ejecuta el navegador.

```javascript
console.log(navigator.platform); // Muestra el sistema operativo
```

### navigator.language

Devuelve la configuración del idioma preferido del navegador.

```javascript
console.log(navigator.language); // Muestra el idioma preferido
```

### navigator.cookieEnabled

Devuelve un valor booleano que indica si las cookies están habilitadas en el navegador.

```javascript
console.log(navigator.cookieEnabled); // Muestra si las cookies están habilitadas
```

## Ejemplos de Uso del Objeto navigator

Aquí hay algunos ejemplos que demuestran cómo se puede utilizar el objeto `navigator` para obtener información sobre el navegador y el entorno del usuario.

### Obtener el Nombre del Navegador

```javascript
// Obtener el nombre del navegador
let nombreNavegador = navigator.appName;
console.log(`Navegador: ${nombreNavegador}`);
```

### Obtener el User Agent

```javascript
// Obtener el user agent
let userAgent = navigator.userAgent;
console.log(`User Agent: ${userAgent}`);
```

### Verificar si las Cookies Están Habilitadas

```javascript
// Verificar si las cookies están habilitadas
if (navigator.cookieEnabled) {
  console.log("Las cookies están habilitadas.");
} else {
  console.log("Las cookies no están habilitadas.");
}
```

### Obtener el Idioma Preferido

```javascript
// Obtener el idioma preferido del navegador
let idioma = navigator.language;
console.log(`Idioma preferido: ${idioma}`);
```

## Consideraciones

- Aunque el objeto `navigator` proporciona información útil sobre el navegador y el sistema operativo, no se debe depender de él para realizar funciones críticas en la aplicación. La información del navegador puede ser manipulada y no siempre es precisa.
- Es recomendable utilizar características de detección de características en lugar de la detección de navegador para garantizar la compatibilidad y mejorar la experiencia del usuario.
