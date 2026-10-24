# Tipos de coverage

El code coverage no es un concepto unico. Existen varios niveles de granularidad que revelan distintos aspectos de la exhaustividad de los tests. Cada tipo responde a una pregunta distinta sobre el codigo ejecutado.

## Line coverage

Es el nivel mas basico. Mide cuantas lineas de codigo fuente se han ejecutado al menos una vez. Una linea cuenta como cubierta si se ejecuta cualquier instruccion contenida en ella. Es el tipo mas sencillo de calcular y el que soportan practicamente todas las herramientas.

```java
int x = calcular(algo);   // cubierta si se ejecuta
if (x > 0) {              // cubierta si se evalua la condicion
    return procesar(x);   // cubierta solo si x > 0
}
return -1;                 // cubierta solo si x <= 0
```

## Branch coverage

Evalua si cada rama de una estructura de control (if, switch, while, for) se ha ejecutado en ambas direcciones. En un `if` simple existen dos ramas: verdadero y falso. Tener line coverage del 100 % sobre un `if` no garantiza que se haya probado la rama falsa; branch coverage si lo exige.

```java
if (activo) {   // branch 1: activo == true
    habilitar();
}               // branch 2: activo == false
```

El branch coverage se expresa frecuentemente como el numero de ramas ejecutadas dividido entre el total de ramas. Es mas riguroso que line coverage porque descubre condiciones no probadas.

## Method coverage

Indica que metodos de una clase se han invocado durante los tests. Un metodo puede tener cero line coverage si nunca se llama, pero si se llama y ejecuta todas sus lineas, el method coverage es completo. Es util para detectar codigo muerto o funcionalidad sin ningun tipo de verificacion.

```java
class Servicio {
    void metodoUsado() { ... }      // cubierto
    void metodoNuncaLlamado() { ... } // NO cubierto
}
```

## Condition coverage

Tambien llamado condition/branch coverage. Analiza cada condicion booleana individual dentro de una expresion compuesta. En una condicion `if (a && b)`, el condition coverage requiere probar `a = true/false` y `b = true/false` de forma independiente, no solo el resultado combinado.

```java
if (a && b) {  // condiciones individuales: a, b
    accion();
}
```

Para cubrir completamente las condiciones hacen falta combinaciones que pueden crecer exponencialmente con el numero de operandos.

## Path coverage

Es el nivel mas exhaustivo y el mas dificil de alcanzar. Requiere ejecutar todas las rutas posibles a traves del codigo, es decir, todas las combinaciones secuenciales de ramas. En un programa con bucles y condiciones anidadas el numero de paths puede ser infinito, por lo que en la practica se limita a paths acotados o se utilizan tecnicas de analisis estatico.

```java
void ejemplo(int x, int y) {
    if (x > 0) { s1(); }   // path 1: x>0, y>0
    if (y > 0) { s2(); }   // path 2: x>0, y<=0
}                          // path 3: x<=0, y>0
                           // path 4: x<=0, y<=0
```

## Relacion entre tipos

Cada tipo de coverage subsume al anterior de forma parcial. Branch coverage implica line coverage, pero no al reves. Condition coverage puede estar completo aun con branch coverage parcial si las condiciones se evaluan en cortocircuito. Path coverage es el unico que garantiza que todas las combinaciones de decisiones se han probado, pero su costo computacional lo hace impracticable para codigo real.

En proyectos Java, las herramientas mas usadas (JaCoCo, Cobertura) trabajan con line y branch coverage. Para condition coverage se requieren herramientas especializadas o analisis manual.
