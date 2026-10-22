<h1 align="center">API </h1>

<h2>📑 Contenido</h2>

- [¿Qué es una API?](#qué-es-una-api)
- [Tipos de APIs Web más comunes](#tipos-de-apis-web-más-comunes)
  - [APIs del Navegador (Browser APIs)](#apis-del-navegador-browser-apis)
  - [APIs de Terceros (Third-Party APIs)](#apis-de-terceros-third-party-apis)
  - [APIs del Servidor (Server-Side APIs o REST APIs)](#apis-del-servidor-server-side-apis-o-rest-apis)
- [¿Por qué son tan importantes las APIs?](#por-qué-son-tan-importantes-las-apis)

## ¿Qué es una API?

API son las siglas de Application Programming Interface (Interfaz de Programación de Aplicaciones). En términos sencillos, una API es un conjunto de reglas y protocolos que permite que diferentes aplicaciones de software se comuniquen entre sí. Piensa en ella como un camarero en un restaurante.

- Tú (la aplicación cliente) le pides al camarero (la API) lo que quieres.

- El camarero va a la cocina (el servidor o la base de datos) y hace el pedido.

- El camarero te trae de vuelta el resultado (la respuesta del servidor).

- Tú no necesitas saber cómo funciona la cocina (la implementación interna), solo necesitas saber qué pedir y cómo pedirlo.

Tú no necesitas saber cómo funciona la cocina (la implementación interna), solo necesitas saber qué pedir y cómo pedirlo.

## Tipos de APIs Web más comunes

### APIs del Navegador (Browser APIs)

Son APIs integradas en el navegador web que te permiten interactuar con el entorno del usuario. Son la base de muchas de las funcionalidades de JavaScript en la web.

- DOM API: Te permite manipular la estructura y el contenido de una página web (el HTML).

- Geolocation API: Te permite obtener la ubicación del usuario (con su permiso).

- Web Storage API: Te permite almacenar datos en el navegador del usuario (`localStorage` y `sessionStorage`).

- Fetch API: Te permite realizar peticiones de red para obtener recursos, como datos de otras APIs o archivos.

### APIs de Terceros (Third-Party APIs)

Son APIs que no son parte del navegador y que se utilizan para conectar tu aplicación con servicios de terceros.

- Google Maps API: Te permite integrar mapas de Google en tu sitio.

- Stripe API: Te permite procesar pagos online.

- Twitter API: Te permite acceder a la información de Twitter para mostrar tweets en tu sitio.

### APIs del Servidor (Server-Side APIs o REST APIs)

Son APIs que se ejecutan en un servidor y que permiten a una aplicación cliente (como tu página web) solicitar y enviar datos a un servidor. Siguen un modelo llamado REST (Representational State Transfer), que es un conjunto de principios de diseño. Se basan en el concepto de recursos (como `usuarios`, `productos`, `posts`) y utilizan los métodos HTTP (`GET`, `POST`, `PUT`, `DELETE`) para interactuar con ellos.

- `GET /api/productos`: Obtener una lista de productos.

- `POST /api/productos`: Crear un nuevo producto.

## ¿Por qué son tan importantes las APIs?

- Modularidad: Permiten dividir una aplicación en partes más pequeñas y manejables.

- Reutilización: Puedes usar servicios existentes sin tener que desarrollarlos tú mismo.

- Escalabilidad: Hacen que sea más fácil escalar y mantener las aplicaciones a medida que crecen.
