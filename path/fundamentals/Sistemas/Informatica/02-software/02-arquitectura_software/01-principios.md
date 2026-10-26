# Principios de la arquitectura de software

La arquitectura de software es el conjunto de decisiones de diseño estructural que definen cómo se organizan, comunican y colaboran los componentes de un sistema informático. Estas decisiones tienen un impacto duradero en la calidad del sistema: su mantenibilidad, escalabilidad, rendimiento y facilidad para incorporar cambios futuros. Una buena arquitectura no garantiza el éxito de un proyecto, pero una mala arquitectura prácticamente garantiza su fracaso a largo plazo.

Los principios de la arquitectura de software no son reglas absolutas sino guías que orientan las decisiones de diseño. Su aplicación correcta requiere juicio, experiencia y un entendimiento profundo del contexto del problema. Lo que funciona para un sistema de trading de alta frecuencia puede ser inapropiado para una aplicación móvil de consumo.

## Modularidad

La modularidad es el principio de dividir un sistema en componentes independientes (*módulos*) que se comunican a través de interfaces bien definidas. Cada módulo encapsula una funcionalidad específica y puede ser desarrollado, probado y mantenido de forma independiente.

Los beneficios de la modularidad son:

- **Mantenibilidad:** un cambio en un módulo no afecta a los demás, siempre que la interfaz del módulo se mantenga.
- **Reutilización:** los módulos bien diseñados pueden reutilizarse en diferentes partes del sistema o en diferentes proyectos.
- **Paralelismo de desarrollo:** múltiples equipos pueden trabajar simultáneamente en módulos diferentes sin interferirse.
- **Testabilidad:** los módulos pueden probarse de forma aislada, simplificando la verificación.

El grafo de acoplamiento entre módulos debe ser lo más débil posible: los módulos deben comunicarse a través de interfaces estables y evitar dependencias circulares o referencias directas a implementaciones internas.

## Encapsulamiento

El encapsulamiento es el principio de ocultar los detalles internos de un componente y exponer solo lo necesario a través de una interfaz pública. Un módulo encapsulado puede cambiar su implementación interna sin que los módulos que dependen de él se vean afectados, siempre que la interfaz pública se mantenga.

El encapsulamiento reduce la complejidad cognitiva: un desarrollador que utiliza un módulo no necesita entender cómo funciona internamente, solo qué hace y cómo se usa. Esta abstracción permite construir sistemas complejos donde cada componente se entiende en términos de su comportamiento visible, no de su implementación.

> El encapsulamiento no es solo una cuestión de programación: también se aplica a la arquitectura de sistemas. Un microservicio que expone una API REST encapsula su lógica interna, su base de datos y sus dependencias. Los consumidores del servicio solo necesitan conocer la API, no cómo se implementa internamente.

## Separación de preocupaciones

La separación de preocupaciones (*Separation of Concerns*, SoC) es el principio de dividir un sistema en secciones, cada una de las cuales aborda una preocupación separada e independiente. Cada módulo o capa del sistema se responsabiliza de un aspecto específico de la funcionalidad, sin mezclar responsabilidades.

Un ejemplo clásico es la arquitectura MVC (*Model-View-Controller*):

- **Model:** gestiona los datos y la lógica de negocio.
- **View:** gestiona la presentación visual de los datos al usuario.
- **Controller:** gestiona la entrada del usuario y coordina las respuestas del sistema.

Esta separación permite cambiar la interfaz de usuario (View) sin modificar la lógica de negocio (Model), o cambiar la base de datos sin afectar a la vista. La separación de preocupaciones es la base de prácticamente todos los patrones arquitectónicos modernos.

## La Arquitectura de Capas (Modelo Cebolla)

La arquitectura de capas organiza el sistema en una jerarquía de niveles, donde cada capa se comunica solo con la capa inmediatamente superior e inferior. Este patrón es uno de los más antiguos y extendidos en la ingeniería de software, y proporciona una estructura clara y predecible para sistemas de cualquier complejidad.

```
┌─────────────────────────────┐
│        Aplicaciones         │  ← Nivel más alto
├─────────────────────────────┤
│    Shell / Interfaz de       │
│         Usuario             │
├─────────────────────────────┤
│     Sistema Operativo       │
├─────────────────────────────┤
│          Kernel             │
├─────────────────────────────┤
│         Hardware            │  ← Nivel más bajo
└─────────────────────────────┘
```

**Hardware:** la capa física del sistema, incluyendo procesador, memoria, almacenamiento y dispositivos de E/S.

**Kernel:** el núcleo del sistema operativo, que tiene acceso directo al hardware y gestiona los recursos más fundamentales.

**Sistema Operativo:** la capa que proporciona servicios a las aplicaciones: gestión de procesos, memoria, archivos y redes.

**Shell / Interfaz:** la capa de interacción con el usuario, ya sea una línea de comandos (shell), una interfaz gráfica (GUI) o una API programática.

**Aplicaciones:** la capa más alta, donde residen los programas que el usuario ejecuta directamente.

El flujo de datos y control atraviesa las capas de arriba abajo: una aplicación que necesita leer un archivo pasa la solicitud al sistema operativo, que la delega al kernel, que comunica con el hardware del almacenamiento. La respuesta viaja en sentido inverso, de abajo arriba.

> La arquitectura de capas no es perfecta: puede introducir latencia adicional por las múltiples capas de traducción, y puede ser excesivamente rígida para sistemas que requieren comunicación directa entre componentes de diferentes capas. Los patrones arquitectónicos modernos a menudo combinan capas con otras técnicas como microservicios o arquitectura orientada a eventos.

