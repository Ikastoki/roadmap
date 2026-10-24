# Factory

Factory es un patron creacional que encapsula la logica de creacion de objetos, ocultando al cliente el conocimiento de las clases concretas que se instancian. En lugar de usar `new` directamente, se delega en un metodo factory que decide que clase crear segun el contexto. El patron tiene dos variantes principales: Factory Method y Abstract Factory.

## Factory Method

Factory Method define una interfaz para crear un objeto, pero permite que las subclases decidan que clase instanciar. La creacion se difiere a las subclases.

```java
// Producto
interface Documento {
    void abrir();
    void guardar();
}

// Productos concretos
class Informe implements Documento {
    public void abrir() { System.out.println("Abriendo informe"); }
    public void guardar() { System.out.println("Guardando informe"); }
}

class HojaCalculo implements Documento {
    public void abrir() { System.out.println("Abriendo hoja de calculo"); }
    public void guardar() { System.out.println("Guardando hoja de calculo"); }
}

// Creador abstracto
abstract class Aplicacion {
    public void nuevoDocumento() {
        Documento doc = crearDocumento();
        doc.abrir();
    }

    // Factory Method
    protected abstract Documento crearDocumento();
}

// Creadores concretos
class AplicacionInformes extends Aplicacion {
    @Override
    protected Documento crearDocumento() {
        return new Informe();
    }
}

class AplicacionHojas extends Aplicacion {
    @Override
    protected Documento crearDocumento() {
        return new HojaCalculo();
    }
}
```

El cliente trabaja con la abstraccion `Aplicacion` y no conoce las clases concretas de documentos. El factory method `crearDocumento()` encapsula la decision de que tipo crear.

```java
Aplicacion app = new AplicacionInformes();
app.nuevoDocumento(); // "Abriendo informe"
```

## Abstract Factory

Abstract Factory provee una interfaz para crear familias de objetos relacionados sin especificar sus clases concretas. Es util cuando el sistema debe ser configurado con una de multiples familias de productos.

```java
// Interfaces de producto
interface Boton {
    void renderizar();
}

interface Ventana {
    void mostrar();
}

// Productos concretos para un tema oscuro
class BotonOscuro implements Boton {
    public void renderizar() { System.out.println("Boton oscuro"); }
}

class VentanaOscura implements Ventana {
    public void mostrar() { System.out.println("Ventana oscura"); }
}

// Productos concretos para un tema claro
class BotonClaro implements Boton {
    public void renderizar() { System.out.println("Boton claro"); }
}

class VentanaClara implements Ventana {
    public void mostrar() { System.out.println("Ventana clara"); }
}

// Abstract Factory
interface FabricaTema {
    Boton crearBoton();
    Ventana crearVentana();
}

// Fabricas concretas
class FabricaTemaOscuro implements FabricaTema {
    public Boton crearBoton() { return new BotonOscuro(); }
    public Ventana crearVentana() { return new VentanaOscura(); }
}

class FabricaTemaClaro implements FabricaTema {
    public Boton crearBoton() { return new BotonClaro(); }
    public Ventana crearVentana() { return new VentanaClara(); }
}
```

El cliente recibe una fabrica y crea objetos consistentes con el tema seleccionado:

```java
FabricaTema fabrica = new FabricaTemaOscuro();
Boton boton = fabrica.crearBoton();
Ventana ventana = fabrica.crearVentana();
boton.renderizar();  // "Boton oscuro"
ventana.mostrar();   // "Ventana oscura"
```

La consistencia entre productos es clave: no se mezclan accidentalmente componentes de distintos temas.

## Ejemplos en la API de Java

La API estandar de Java utiliza Factory Method y Abstract Factory ampliamente.

### Calendar.getInstance()

```java
Calendar cal = Calendar.getInstance();
```

No se sabe ni importa que implementacion concreta se devuelve. `Calendar.getInstance()` puede devolver `GregorianCalendar` (la mayoria de los casos) u otras implementaciones especificas segun la locale. El cliente obtiene un `Calendar` funcional sin conocer su tipo real.

### Executors

La clase `Executors` es una fabrica de `ExecutorService`:

```java
ExecutorService poolFijo = Executors.newFixedThreadPool(10);
ExecutorService poolDinamico = Executors.newCachedThreadPool();
ExecutorService unHilo = Executors.newSingleThreadExecutor();
ScheduledExecutorService programado = Executors.newScheduledThreadPool(5);
```

Cada metodo factory crea una configuracion especifica de `ThreadPoolExecutor` con parametros distintos, ocultando la complejidad de su construccion.

### Pattern.compile()

```java
Pattern patron = Pattern.compile("\\d+");
Matcher matcher = patron.matcher(texto);
```

`Pattern.compile()` es un factory method que puede devolver implementaciones internas optimizadas segun el patron, sin que el cliente lo perciba.

### Collections

`Collections` proporciona metodos factory estaticos para crear colecciones inmutables y vistas sincronizadas:

```java
List<String> inmutable = Collections.unmodifiableList(lista);
List<String> sincronizada = Collections.synchronizedList(lista);
Set<String> singleton = Collections.singleton("unico");
```

> El patron Factory simplifica el mantenimiento: cuando aparece una nueva variante de un producto, solo se anade una nueva clase concreta y se modifica la fabrica (o se registra una nueva fabrica). El resto del sistema permanece inalterado porque depende de las abstracciones, no de las implementaciones concretas.
