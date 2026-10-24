# Otras colecciones

Ademas de las implementaciones principales del framework de colecciones, Java incluye clases legacy y especializadas para casos de uso concretos. Algunas estan obsoletas; otras son altamente especificas y utiles en contextos determinados.

## Vector

`Vector` es una clase legacy (Java 1.0) que implementa una lista dinamica similar a `ArrayList`, pero con todos sus metodos sincronizados.

```java
Vector<String> vector = new Vector<>();
vector.add("A");
vector.addElement("B"); // metodo legacy
String primero = vector.firstElement();
String ultimo = vector.lastElement();
int capacidad = vector.capacity();
```

### Diferencias con ArrayList

- Sincronizacion: todos los metodos son `synchronized`.
- Crecimiento: duplica su capacidad cuando se llena; `ArrayList` crece un 50%.
- Metodos legacy: `addElement()`, `removeElement()`, `elementAt()`, `firstElement()`, `lastElement()`, `capacity()`, `setSize()`.

```java
// Capacity increment especifico
Vector<String> v = new Vector<>(10, 5); // inicial 10, incremento 5
```

`Vector` esta practicamente obsoleto. Para uso monohilo use `ArrayList`; para concurrencia use `Collections.synchronizedList(new ArrayList<>())` o `CopyOnWriteArrayList`.

## Stack

`Stack` extiende `Vector` y modela una pila LIFO (last-in-first-out).

```java
Stack<String> pila = new Stack<>();
pila.push("A");
pila.push("B");
String cima = pila.peek(); // "B"
String extraido = pila.pop(); // "B"
int pos = pila.search("A"); // 2 (posicion desde cima, 1-indexed)
boolean vacia = pila.empty();
```

### Problemas de Stack

- Extiende `Vector`, por lo que hereda metodos que no pertenecen a una pila (`add(int, E)`, `remove(int)`, `elementAt()`), rompiendo la abstraccion LIFO.
- Es sincronizado (hereda de `Vector`), con sobrecarga innecesaria.
- `empty()` y `search()` son metodos legacy.

La alternativa moderna es `ArrayDeque`:

```java
Deque<String> pilaModerna = new ArrayDeque<>();
pilaModerna.push("A");
pilaModerna.push("B");
String cima = pilaModerna.peek();
String extraido = pilaModerna.pop();
```

## Properties

`Properties` extiende `Hashtable<Object, Object>` y esta disenado para manejar archivos de configuracion clave-valor (tipicamente cadenas).

```java
Properties props = new Properties();
props.setProperty("url", "jdbc:mysql://localhost:3306/db");
props.setProperty("user", "admin");

String url = props.getProperty("url");
String port = props.getProperty("port", "3306"); // con valor por defecto
```

### Carga y guardado de archivos

```java
// Cargar desde archivo
Properties props = new Properties();
try (FileInputStream fis = new FileInputStream("config.properties")) {
    props.load(fis);
} catch (IOException e) {
    e.printStackTrace();
}

// Guardar a archivo
try (FileOutputStream fos = new FileOutputStream("config.properties")) {
    props.store(fos, "Configuracion de la aplicacion");
} catch (IOException e) {
    e.printStackTrace();
}
```

```java
// Cargar desde XML
props.loadFromXML(new FileInputStream("config.xml"));
// Guardar como XML
props.storeToXML(new FileOutputStream("config.xml"), "comentario");
```

## EnumSet

`EnumSet` es una implementacion de `Set` altamente eficiente para tipos enumerados. Internamente usa un vector de bits (bit field) en lugar de un `HashSet`.

```java
enum Dia { LUNES, MARTES, MIERCOLES, JUEVES, VIERNES, SABADO, DOMINGO }

EnumSet<Dia> laborables = EnumSet.range(Dia.LUNES, Dia.VIERNES);
EnumSet<Dia> finDeSemana = EnumSet.of(Dia.SABADO, Dia.DOMINGO);
EnumSet<Dia> todos = EnumSet.allOf(Dia.class);
EnumSet<Dia> ninguno = EnumSet.noneOf(Dia.class);
```

### Rendimiento

- `add()`, `remove()`, `contains()`: O(1) (operaciones bit a bit).
- Consumo de memoria minimo: un `long` para enums de hasta 64 valores.
- Iteracion en orden natural del enum.

```java
EnumSet<Dia> dias = EnumSet.complementOf(laborables); // SABADO, DOMINGO
```

## EnumMap

`EnumMap` es un `Map` especializado para claves de tipo enum. Internamente usa un array indexado por ordinal del enum.

```java
enum Color { ROJO, VERDE, AZUL }

EnumMap<Color, String> mapa = new EnumMap<>(Color.class);
mapa.put(Color.ROJO, "#FF0000");
mapa.put(Color.VERDE, "#00FF00");
mapa.put(Color.AZUL, "#0000FF");
```

### Rendimiento

- `put()`, `get()`, `remove()`: O(1) (acceso directo a array).
- Sin colisiones de hash.
- Iteracion en orden natural del enum.

## WeakHashMap

`WeakHashMap` usa referencias debiles (weak references) para las claves. Cuando una clave ya no tiene referencias fuertes externas, el GC puede recolectarla y la entrada se eliminara del mapa en la siguiente operacion.

```java
WeakHashMap<Object, String> weakMap = new WeakHashMap<>();
Object key = new Object();
weakMap.put(key, "valor");

// Si key se queda sin referencias fuertes...
key = null;
System.gc(); // ...la entrada puede ser recolectada
```

Util para caches o almacenamiento de metadatos asociados a objetos que no deben impedir la recoleccion de basura.

```java
// Cache de metadatos asociados a objetos temporales
WeakHashMap<Object, Metadata> metadatosCache = new WeakHashMap<>();
```

No se debe usar para claves de tipo `String` o literales, porque las cadenas literales tienen referencias fuertes en el String pool.

## IdentityHashMap

`IdentityHashMap` usa igualdad de referencia (`==`) en lugar de `equals()` para comparar claves. Tambien usa direccionamiento lineal (linear probing) en lugar de buckets hash.

```java
IdentityHashMap<String, String> identityMap = new IdentityHashMap<>();

String a = new String("clave");
String b = new String("clave");

identityMap.put(a, "primero");
identityMap.put(b, "segundo");

System.out.println(identityMap.size()); // 2 (a y b son referencias distintas)
```

Util para:

- Contar referencias a objetos (profiling).
- Mantener mapas de objetos donde la identidad importa (serializacion, proxies).
- Implementar algoritmos que necesitan igualdad de referencia.
