# Patrones de diseno

Los patrones de diseno son soluciones recurrentes a problemas comunes en el desarrollo de software. No son fragmentos de codigo listos para copiar, sino descripciones de como estructurar clases y objetos para resolver un problema en un contexto determinado. Cada patron define un nombre, el problema que resuelve, la solucion propuesta, y las consecuencias de aplicarlo.

Los patrones fueron catalogados y formalizados en el libro _Design Patterns: Elements of Reusable Object-Oriented Software_ (1994), conocido como el libro de la _Gang of Four_ (GoF). Erich Gamma, Richard Helm, Ralph Johnson y John Vlissides identificaron 23 patrones clasificados en tres categorias segun su proposito.

## Categorias de patrones

### Creacionales

Los patrones creacionales abstraen el proceso de instanciacion, ocultando la logica de creacion y haciendo que el sistema sea independiente de como se crean, componen y representan sus objetos.

- **Singleton**: garantiza una unica instancia de una clase.
- **Factory Method**: define una interfaz para crear objetos, delegando en subclases la decision de que clase instanciar.
- **Abstract Factory**: provee una interfaz para crear familias de objetos relacionados sin especificar sus clases concretas.
- **Builder**: separa la construccion de un objeto complejo de su representacion.
- **Prototype**: crea nuevos objetos clonando una instancia existente.

### Estructurales

Los patrones estructurales se centran en como combinar clases y objetos para formar estructuras mas grandes, manteniendo flexibilidad y eficiencia.

- **Adapter**: permite que clases con interfaces incompatibles trabajen juntas.
- **Decorator**: anade responsabilidades a un objeto dinamicamente.
- **Composite**: compone objetos en estructuras de arbol para representar jerarquias parte-todo.
- **Proxy**: proporciona un sustituto o representante de otro objeto para controlar el acceso.
- **Facade**: ofrece una interfaz simplificada a un subsistema complejo.
- **Bridge**: desacopla una abstraccion de su implementacion.
- **Flyweight**: comparte objetos pequeños para soportar grandes cantidades de ellos eficientemente.

### De comportamiento

Los patrones de comportamiento tratan la interaccion y distribucion de responsabilidades entre objetos.

- **Observer**: define una dependencia uno-a-muchos entre objetos para que cuando uno cambie su estado, todos los dependientes sean notificados.
- **Strategy**: define una familia de algoritmos intercambiables.
- **Template Method**: define el esqueleto de un algoritmo en un metodo, delegando pasos a subclases.
- **Command**: encapsula una solicitud como un objeto.
- **Iterator**: proporciona una forma de acceder secuencialmente a los elementos de una coleccion sin exponer su representacion interna.
- **State**: permite que un objeto altere su comportamiento cuando cambia su estado interno.
- **Chain of Responsibility**: pasa una solicitud a traves de una cadena de manejadores.
- **Mediator**: define un objeto que centraliza la comunicacion entre objetos.
- **Memento**: captura y externaliza el estado interno de un objeto para restaurarlo despues.
- **Visitor**: separa un algoritmo de la estructura de objetos sobre la que opera.

## Cuando usar patrones

Los patrones no deben aplicarse indiscriminadamente. Son herramientas para situaciones especificas. Una aplicacion forzada de patrones donde no se necesitan produce sobrediseno, complejidad innecesaria y codigo dificil de mantener.

La decision de usar un patron debe basarse en:

- **Problemas recurrentes**: el patron resuelve un problema que ya has encontrado antes.
- **Vocabulario comun**: el patron proporciona un nombre que facilita la comunicacion entre desarrolladores.
- **Flexibilidad futura**: el patron anticipa cambios que probablemente ocurriran.
- **Compensaciones**: cada patron tiene costos (complejidad, indireccion, rendimiento) que deben evaluarse.

```java
// Sin patron: instanciacion directa fragil
Pedido pedido = new Pedido();
pedido.setTipo(new EnvioInternacional());
pedido.setPago(new PagoTarjeta());

// Con Abstract Factory: familias intercambiables
FabricaEnvio fabrica = obtenerFabrica(pais);
Pedido pedido = new Pedido(fabrica);
```

## Anti-patrones

Un antipatron es una solucion que parece apropiada pero tiene consecuencias negativas. Es el equivalente a un "mal patron".

- **God Class**: una clase que acumula demasiada responsabilidad.
- **Spaghetti Code**: codigo sin estructura clara, con flujos de control enmarañados.
- **Singleton abusado**: usar Singleton cuando se necesita una instancia normal o cuando introduce acoplamiento global.
- **Yo-Yo Problem**: jerarquias de herencia excesivamente profundas que obligan a saltar entre multiples niveles para entender el flujo.
- **Golden Hammer**: aplicar un patron conocido a todo problema, aunque no sea adecuado.

> La identificacion de antipatrones es tan importante como el conocimiento de los patrones. Reconocer una mala solucion permite refactorizarla hacia un diseno mas solido antes de que el coste de correccion sea prohibitivo.

Los patrones de diseno son parte del lenguaje de los desarrolladores experimentados. Conocerlos no solo proporciona soluciones probadas, sino que permite abstraer el diseno a un nivel superior, donde la conversacion ya no trata de clases concretas sino de las relaciones e intenciones entre ellas.
