# Que es la arquitectura de capas

Todo sistema software crece. Lo que comienza como un punado de clases y metodos termina, con el tiempo, convirtiendose en una marana de dependencias donde cada componente conoce demasiado sobre los demas. La arquitectura de capas (layered architecture) propone un antídoto estructurado: organizar el codigo en niveles horizontales superpuestos, donde cada nivel asume una responsabilidad concreta y se comunica exclusivamente con el nivel inmediatamente inferior. Es, con diferencia, el patron arquitectonico mas extendido en el ecosistema Java empresarial, y constituye la base sobre la que se asientan Spring MVC, Jakarta EE y la mayoria de frameworks corporativos.

El patron prescribe que todo el software de una aplicacion se divide en capas, tipicamente tres: presentacion, logica de negocio y persistencia. La capa de presentacion se ocupa de la interaccion con el usuario --controladores HTTP, interfaces graficas, APIs REST-- sin contener ninguna decision de negocio. La capa de negocio (o servicio) contiene las reglas y procesos que dan valor a la aplicacion: calculos, validaciones, orquestacion de operaciones. La capa de persistencia gestiona el almacenamiento y recuperacion de datos, ya sea contra una base de datos relacional, un sistema de archivos o un servicio externo.

## Separacion de responsabilidades

El principio fundamental que sostiene este modelo es la separacion de responsabilidades (Separation of Concerns, SoC). Cada capa resuelve un problema distinto y no debe mezclar logicas que pertenezcan a otra. Un controlador HTTP no deberia construir consultas SQL; un repositorio no deberia decidir si un usuario tiene permiso para ejecutar una operacion. Esta separacion no es una cuestion de estilo: es una barrera estructural que limita el impacto de los cambios. Cuando un requisito de persistencia cambia --por ejemplo, al migrar de MySQL a MongoDB-- solo la capa de persistencia debe modificarse, siempre que los contratos (interfaces) entre capas se mantengan estables.

La implementacion en Java suele apoyarse en interfaces para definir los contratos entre capas. La capa superior depende de una abstraccion (una interfaz) y la capa inferior proporciona una implementacion concreta. Esto permite sustituir implementaciones sin alterar el codigo cliente, y habilita tecnicas como la inyeccion de dependencias que Spring materializa a traves de su contenedor IoC.

> La SoC no implica que las capas sean completamente ignorantes entre si; significa que cada capa conoce una abstraccion de la capa inferior, no sus detalles internos.

## Comunicacion entre capas

El flujo de comunicacion es estrictamente unidireccional: la capa superior invoca a la capa inferior, nunca al reves. Un servicio llama a un repositorio, pero el repositorio no llama al servicio. Un controlador llama a un servicio, pero el servicio no llama al controlador. Esta direccionalidad unica evita dependencias ciclicas y mantiene el grafo de dependencias aciclico, lo que simplifica el testing, el despliegue y la comprension del sistema.

El mecanismo tipico de comunicacion es la invocacion sincrona de metodos: la capa A llama a un metodo publico de la capa B y espera el resultado. En Java, esto se traduce en llamadas a objetos cuyas referencias se obtienen a traves del contenedor de dependencias. Sin embargo, el patron no exige sincronia; variantes del mismo admiten colas de mensajes, eventos o llamadas asincronas siempre que se respete la direccionalidad.

> Una consecuencia sutil pero importante: la capa de negocio no sabe si la entrada le llega de un controlador HTTP, un cliente de escritorio o una prueba unitaria. Del mismo modo, la capa de persistencia ignora las reglas de negocio que motivaron el almacenamiento de un dato. Esta ignorancia es, paradójicamente, la mayor fortaleza del patron.
