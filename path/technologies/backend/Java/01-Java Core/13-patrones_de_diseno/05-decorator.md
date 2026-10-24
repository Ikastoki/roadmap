# Decorator

Decorator es un patron estructural que permite anadir responsabilidades a un objeto de forma dinamica, sin modificar su clase ni recurrir a la herencia. En lugar de extender una clase para anadir comportamiento, se "envuelve" el objeto original con uno o mas decoradores que añaden funcionalidad antes o despues de delegar en el objeto envuelto.

## Envolver objetos para comportamiento dinamico

La idea central es que tanto el componente base como los decoradores implementan la misma interfaz. El decorador mantiene una referencia al componente que envuelve y delega en el antes o despues de ejecutar su propia logica. Esto permite apilar multiples decoradores, cada uno anadiendo una capa de comportamiento.

```java
// Interfaz del componente
interface Cafe {
    double getCosto();
    String getDescripcion();
}

// Componente concreto
class CafeBasico implements Cafe {
    @Override
    public double getCosto() {
        return 2.0;
    }

    @Override
    public String getDescripcion() {
        return "Cafe basico";
    }
}

// Decorador abstracto
abstract class DecoradorCafe implements Cafe {
    protected Cafe cafeDecorado;

    public DecoradorCafe(Cafe cafe) {
        this.cafeDecorado = cafe;
    }

    @Override
    public double getCosto() {
        return cafeDecorado.getCosto();
    }

    @Override
    public String getDescripcion() {
        return cafeDecorado.getDescripcion();
    }
}

// Decoradores concretos
class ConLeche extends DecoradorCafe {
    public ConLeche(Cafe cafe) {
        super(cafe);
    }

    @Override
    public double getCosto() {
        return super.getCosto() + 0.5;
    }

    @Override
    public String getDescripcion() {
        return super.getDescripcion() + ", con leche";
    }
}

class ConAzucar extends DecoradorCafe {
    public ConAzucar(Cafe cafe) {
        super(cafe);
    }

    @Override
    public double getCosto() {
        return super.getCosto() + 0.3;
    }

    @Override
    public String getDescripcion() {
        return super.getDescripcion() + ", con azucar";
    }
}

class ConCrema extends DecoradorCafe {
    public ConCrema(Cafe cafe) {
        super(cafe);
    }

    @Override
    public double getCosto() {
        return super.getCosto() + 1.0;
    }

    @Override
    public String getDescripcion() {
        return super.getDescripcion() + ", con crema";
    }
}
```

El resultado es una composicion flexible en tiempo de ejecucion:

```java
Cafe cafe = new CafeBasico();
System.out.println(cafe.getDescripcion() + " -> " + cafe.getCosto());
// "Cafe basico -> 2.0"

cafe = new ConLeche(cafe);
System.out.println(cafe.getDescripcion() + " -> " + cafe.getCosto());
// "Cafe basico, con leche -> 2.5"

cafe = new ConAzucar(cafe);
System.out.println(cafe.getDescripcion() + " -> " + cafe.getCosto());
// "Cafe basico, con leche, con azucar -> 2.8"

// Decoracion directa sin variable intermedia
Cafe cafeCompleto = new ConCrema(new ConAzucar(new ConLeche(new CafeBasico())));
System.out.println(cafeCompleto.getDescripcion() + " -> " + cafeCompleto.getCosto());
// "Cafe basico, con leche, con azucar, con crema -> 3.5"
```

Para conseguir la misma combinacion con herencia habria que crear clases como `CafeConLecheYAzucar`, `CafeConLecheYCrema`, `CafeConLecheAzucarYCrema`, etc. El numero de clases crece exponencialmente. Con Decorator, cada opcion es una clase independiente que se compone dinamicamente.

## Ventajas y consideraciones

- **Mas flexible que la herencia**: permite combinar comportamientos en cualquier orden y cantidad.
- **Principio de responsabilidad unica**: cada decorador se centra en una sola responsabilidad.
- **Composicion en tiempo de ejecucion**: las combinaciones se deciden en ejecucion, no en compilacion.
- **Transparencia**: el cliente trata al objeto decorado como si fuera el componente original.

Como desventaja, el uso excesivo de decoradores puede producir muchos objetos pequeños y una pila de llamadas profunda. La depuracion se vuelve mas compleja porque el flujo atraviesa multiples capas.

## Ejemplos en la API de Java

### BufferedReader

`BufferedReader` es un decorador que anade buffer a un `Reader`:

```java
Reader lector = new BufferedReader(new FileReader("archivo.txt"));
```

```java
Reader lector = new LineNumberReader(
    new BufferedReader(
        new FileReader("archivo.txt")
    )
);
```

Cada capa anade funcionalidad: `FileReader` lee bytes del archivo, `BufferedReader` anade buffer para eficiencia, `LineNumberReader` anade seguimiento de lineas.

### Collections.synchronizedList

Los metodos de `Collections` envuelven colecciones con decoradores sincronizados:

```java
List<String> lista = new ArrayList<>();
List<String> listaSincronizada = Collections.synchronizedList(lista);
// Ahora cada operacion esta sincronizada
```

`Collections.unmodifiableList` es otro decorador que anade proteccion contra modificacion:

```java
List<String> soloLectura = Collections.unmodifiableList(lista);
soloLectura.add("nuevo"); // Lanza UnsupportedOperationException
```

### InputStream

La jerarquia de `java.io.InputStream` es un ejemplo clasico del patron:

```java
InputStream flujo = new GZIPInputStream(
    new BufferedInputStream(
        new FileInputStream("datos.gz")
    )
);
```

`FileInputStream` lee bytes del archivo. `BufferedInputStream` anade buffer. `GZIPInputStream` anade descompresion. Cada decorador es intercambiable y combinable.

> Decorator es uno de los patrones mas utilizados en la biblioteca estandar de Java. Reconocerlo ayuda a entender como se combinan funcionalidades en las APIs de I/O, colecciones y streams, y a disenar sistemas propios con la misma flexibilidad.
