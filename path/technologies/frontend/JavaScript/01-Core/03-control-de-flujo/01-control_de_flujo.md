<h1 align='center'>Control de Flujo</h1>

<h2>📑 Contenido</h2>

- [Control de Flujo](#control-de-flujo)
- [¿Por qué es importante el Control de Flujo?](#por-qué-es-importante-el-control-de-flujo)
- [Componentes Clave del Control de Flujo](#componentes-clave-del-control-de-flujo)
- [Ejemplo de Flujo de Ejecución Básico](#ejemplo-de-flujo-de-ejecución-básico)

## Control de Flujo

En programación, el control de flujo se refiere a cómo se organiza y ejecuta el código a lo largo de un programa. En lugar de ejecutarse de forma lineal (de arriba a abajo), el flujo del programa puede ser dirigido o modificado mediante diferentes estructuras. Estas estructuras permiten tomar decisiones, repetir acciones y manejar situaciones excepcionales, lo que brinda a los desarrolladores un alto grado de flexibilidad para resolver problemas complejos de manera eficiente.

JavaScript, al igual que otros lenguajes de programación, incluye varias herramientas que permiten controlar el flujo de ejecución. Mediante el uso de estas herramientas, es posible tomar rutas diferentes en función de condiciones específicas, iterar sobre colecciones de datos o interrumpir la ejecución cuando se cumplen ciertos criterios.

## ¿Por qué es importante el Control de Flujo?

Sin control de flujo, el código se ejecutaría siempre de manera secuencial, haciendo imposible realizar tareas dinámicas o interactivas. Imagina un programa que siempre hace lo mismo, sin importar las entradas o circunstancias: sería extremadamente limitado. El control de flujo te permite hacer que el programa responda a distintas situaciones, como:

Tomar decisiones basadas en entradas del usuario o datos procesados.
Repetir tareas múltiples veces, lo cual es fundamental para automatizar procesos.
Gestionar eventos inesperados, como errores que podrían ocurrir durante la ejecución del programa.

## Componentes Clave del Control de Flujo

El control de flujo en JavaScript se divide en varias categorías principales, que se utilizan para distintas situaciones en un programa:

- **Condicionales:** Permiten tomar decisiones basadas en condiciones. Según si una condición es verdadera o falsa, el programa ejecutará diferentes bloques de código.
- **Bucles:** Proveen un mecanismo para repetir una serie de instrucciones múltiples veces hasta que se cumpla una condición.
- **Manejo de excepciones:** Permite capturar y manejar errores que ocurren durante la ejecución de un programa, evitando que éste se detenga de forma abrupta.
  En esencia, estas estructuras permiten que tu código sea más flexible, dinámico y adaptado a diferentes escenarios.

## Ejemplo de Flujo de Ejecución Básico

Imagina un caso simple en el que tu programa recibe la entrada de un usuario. Dependiendo del valor de esa entrada, el programa decide qué hacer a continuación:

1. El programa pide al usuario que ingrese un número.
1. Si el número es mayor que 10, muestra un mensaje de "número alto".
1. Si el número es menor o igual a 10, muestra un mensaje de "número bajo".
1. El programa se detiene.

Este es un ejemplo básico de cómo la condicionalidad cambia el flujo del programa, haciendo que ejecute diferentes bloques de código según los datos proporcionados.
