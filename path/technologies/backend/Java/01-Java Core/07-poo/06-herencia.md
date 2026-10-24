# Herencia

La herencia es un mecanismo que permite a una clase (subclase) derivar de otra (superclase), heredando sus campos y metodos. En Java, la herencia se implementa con la palabra clave `extends` y sigue un modelo de herencia unica: una clase solo puede extender directamente a una superclase.

## extends

La sintaxis `class Subclase extends Superclase` establece la relacion de herencia. La subclase obtiene todos los miembros `public` y `protected` de la superclase, y opcionalmente puede sobrescribir metodos o anadir nuevos.

```java
public class Animal {
    protected String nombre;

    public Animal(String nombre) {
        this.nombre = nombre;
    }

    public void comer() {
        System.out.println(nombre + " esta comiendo");
    }

    public void dormir() {
        System.out.println(nombre + " esta durmiendo");
    }
}

public class Perro extends Animal {
    private String raza;

    public Perro(String nombre, String raza) {
        super(nombre); // llama al constructor de Animal
        this.raza = raza;
    }

    public void ladrar() {
        System.out.println(nombre + " dice: Guau!");
    }

    @Override
    public void comer() {
        System.out.println(nombre + " come croquetas");
    }
}
```

```java
Perro perro = new Perro("Rex", "Pastor Aleman");
perro.comer();   // "Rex come croquetas" (sobrescrito)
perro.dormir();  // "Rex esta durmiendo" (heredado)
perro.ladrar();  // "Rex dice: Guau!" (propio)
```

## La palabra clave super

`super` permite acceder a miembros de la superclase desde la subclase. Se usa para:

- Invocar al constructor de la superclase: `super(parametros)`
- Acceder a metodos sobrescritos: `super.metodo()`
- Acceder a campos de la superclase: `super.campo`

```java
public class Derivada extends Base {
    public Derivada() {
        super(); // llamada al constructor de Base
    }

    @Override
    public void metodo() {
        super.metodo(); // invoca la version de la superclase
        System.out.println("Extendido en subclase");
    }
}
```

La llamada a `super()` debe ser la primera sentencia del constructor. Si no se escribe explicitamente, el compilador inserta `super()` (el constructor sin argumentos de la superclase). Si la superclase no tiene constructor sin argumentos, es obligatorio llamar explicitamente a `super(parametros)`.

## Jerarquia de herencia

Todas las clases en Java heredan directa o indirectamente de `java.lang.Object`. Esto forma una jerarquia unificada donde cada clase tiene exactamente una superclase (excepto `Object` que no tiene ninguna).

```
Object
  |
  +-- Animal
  |     |
  |     +-- Perro
  |     +-- Gato
  |
  +-- Vehiculo
  |     |
  |     +-- Coche
  |     +-- Moto
  |
  +-- Comparable (interfaz, no clase)
```

La profundidad de la jerarquia no tiene limites teoricos, pero jerarquias muy profundas (> 5 niveles) son sintoma de diseno excesivamente acoplado y dificil de mantener.

Una clase puede comprobar su posicion en la jerarquia con `instanceof`:

```java
Animal a = new Perro("Rex", "Pastor");
System.out.println(a instanceof Perro);  // true
System.out.println(a instanceof Animal); // true
System.out.println(a instanceof Object); // true
```

## Constructor chaining en herencia

Cuando se crea un objeto, los constructores se ejecutan en cadena desde la superclase mas alta hasta la subclase mas baja. Cada constructor llama al de su superclase antes de ejecutar su propio cuerpo.

```java
public class A {
    public A() {
        System.out.println("Constructor A");
    }
}

public class B extends A {
    public B() {
        System.out.println("Constructor B");
    }
}

public class C extends B {
    public C() {
        System.out.println("Constructor C");
    }
}

new C();
// Salida:
// Constructor A
// Constructor B
// Constructor C
```

Este orden garantiza que la parte de la superclase se inicialice antes que la de la subclase, evitando que la subclase acceda a estado no inicializado de su superclase.

## Composicion vs herencia

La herencia modela una relacion "es un" (is-a): un `Perro` es un `Animal`. La composicion modela una relacion "tiene un" (has-a): un `Coche` tiene un `Motor`.

```java
// Herencia: "es un"
public class Coche extends Vehiculo { }

// Composicion: "tiene un"
public class Coche {
    private Motor motor;
    private Rueda[] ruedas;
}
```

La composicion es mas flexible que la herencia porque no crea acoplamiento fuerte entre clases. Los cambios en la superclase afectan a todas las subclases, mientras que en composicion los componentes pueden intercambiarse sin afectar al contenedor.

## Cuando preferir composicion

La herencia debe preferirse solo cuando existe una relacion jerarquica genuina "es un" y la subclase realmente es un subtipo de la superclase (principio de sustitucion de Liskov). En caso de duda, la composicion es la opcion mas segura.

```java
// MAL: herencia injustificada
public class Pila extends ArrayList<String> {
    public void push(String s) { add(s); }
    public String pop() { return remove(size() - 1); }
}
// Pila hereda metodos como add(int, String) que rompen la semantica LIFO

// BIEN: composicion
public class Pila {
    private final List<String> elementos = new ArrayList<>();

    public void push(String s) {
        elementos.add(s);
    }

    public String pop() {
        if (elementos.isEmpty()) throw new EmptyStackException();
        return elementos.remove(elementos.size() - 1);
    }
}
```

La herencia rompe la encapsulacion: las subclases dependen de los detalles de implementacion de la superclase. Si la superclase cambia, las subclases pueden romperse sin que se haya tocado su codigo. La composicion evita este problema al acceder solo a traves de la interfaz publica.

> El principio "favorecer la composicion sobre la herencia" (Gamma et al., Design Patterns) es una de las recomendaciones mas citadas en ingenieria de software. La herencia debe usarse con criterio, no como primera opcion para reutilizar codigo.

## Reglas de la herencia

- Una clase solo puede extender una superclase (herencia unica)
- Una interfaz puede extender multiples interfaces
- Los constructores no se heredan
- Los miembros `private` no son visibles en la subclase, pero existen en el objeto
- Los miembros `final` no pueden sobrescribirse
- Las clases `final` no pueden extenderse
- Toda clase tiene exactamente una superclase (Object si no se especifica)
