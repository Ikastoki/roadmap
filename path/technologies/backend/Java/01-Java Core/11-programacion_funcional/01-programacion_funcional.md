# Programacion funcional

El paradigma funcional concibe la computacion como la evaluacion de funciones matematicas, evitando el estado mutable y los efectos secundarios que caracterizan a la programacion imperativa. Mientras que en POO los objetos encapsulan estado y comportamiento que muta a lo largo del tiempo, la programacion funcional trabaja con valores inmutables y operaciones que producen nuevos valores sin alterar los originales. Java, a partir de su version 8, incorporo elementos funcionales sin abandonar su naturaleza orientada a objetos, dando lugar a un modelo hibrido donde coexisten ambos paradigmas.

## Funciones puras

Una funcion pura es aquella cuyo resultado depende exclusivamente de sus argumentos y que no produce efectos observables en el exterior. Dado el mismo conjunto de entradas, produce siempre la misma salida. Esta propiedad, conocida como transparencia referencial, permite razonar sobre el codigo de manera local: para entender que hace una llamada a una funcion pura basta con examinar sus argumentos.

```java
public int sumar(int a, int b) {
    return a + b;
}
```

Esta funcion es pura. No depende de variables externas, no modifica estado global, no imprime nada, no lanza excepciones. En contraste, el siguiente metodo no es puro porque depende de una variable mutable externa:

```java
private int acumulador = 0;

public int acumular(int valor) {
    acumulador += valor;
    return acumulador;
}
```

## Inmutabilidad

La inmutabilidad es un pilar del paradigma funcional. Un objeto inmutable no puede modificarse una vez creado; cualquier "modificacion" produce una nueva instancia. En Java, esto se consigue declarando todos los campos como `final`, sin metodos setter, y garantizando que ningun metodo devuelva referencias a objetos mutables internos.

```java
public final class Punto {
    private final int x;
    private final int y;

    public Punto(int x, int y) {
        this.x = x;
        this.y = y;
    }

    public Punto desplazar(int dx, int dy) {
        return new Punto(this.x + dx, this.y + dy);
    }

    public int getX() { return x; }
    public int getY() { return y; }
}
```

Trabajar con inmutabilidad elimina clases enteras de errores relacionados con la modificacion accidental del estado compartido en entornos concurrentes.

## Funciones de orden superior

Una funcion de orden superior es aquella que recibe una funcion como argumento, devuelve una funcion, o ambas cosas. En Java, antes de las lambdas esto se lograba con clases anonimas y la implementacion de interfaces funcionales como `Comparator`, `Runnable` o `ActionListener`.

```java
List<String> nombres = Arrays.asList("Ana", "Pedro", "Luis");
Collections.sort(nombres, (a, b) -> a.compareToIgnoreCase(b));
```

El metodo `sort` recibe un `Comparator`, que es una interfaz funcional. La expresion lambda actua como funcion de orden superior que define la estrategia de comparacion. La API de streams esta construida sobre este principio: `map`, `filter`, `reduce` reciben funciones que describen la transformacion a aplicar.

## Expresividad

La programacion funcional favorece el _que_ sobre el _como_. En lugar de escribir bucles imperativos que describen cada paso del algoritmo, se declara la intencion mediante operaciones compuestas. Comparar el enfoque imperativo:

```java
List<String> resultado = new ArrayList<>();
for (String s : lista) {
    if (s.startsWith("A")) {
        resultado.add(s.toUpperCase());
    }
}
```

Frente al enfoque funcional:

```java
List<String> resultado = lista.stream()
    .filter(s -> s.startsWith("A"))
    .map(String::toUpperCase)
    .collect(Collectors.toList());
```

La version funcional expresa la intencion directamente: filtrar, transformar y recolectar. No hay variables de control, no hay mutacion explicita, no hay detalles de iteracion. El codigo resultante es mas declarativo, legible y menos propenso a errores.

## Programacion funcional vs POO

POO y programacion funcional no son excluyentes. Java los combina. La POO proporciona una forma natural de modelar el dominio mediante tipos y relaciones de herencia; la programacion funcional ofrece herramientas poderosas para el procesamiento de datos y la composicion de comportamientos. En la practica, un diseno solido usa objetos para estructurar el sistema y funciones para las operaciones sobre los datos.

> La programacion funcional no busca reemplazar la POO, sino complementarla. La clave esta en reconocer que problema se beneficia de cada paradigma y aplicar el adecuado sin dogmatismo.

Las colecciones inmutables (`List.copyOf`, `Set.of`, `Map.ofEntry`) y los records (`record Punto(int x, int y) {}`) introducidos en versiones recientes de Java refuerzan el estilo funcional al facilitar la creacion de objetos de datos inmutables con minima ceremonia.
