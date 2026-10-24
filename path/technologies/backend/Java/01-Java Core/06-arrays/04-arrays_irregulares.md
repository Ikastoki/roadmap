# Arrays irregulares

Un array irregular (jagged array) es un array multidimensional donde las filas tienen longitudes diferentes. A diferencia de otros lenguajes donde las matrices son rectangulares por definicion, en Java los arrays multidimensionales son arrays de arrays, lo que permite que cada fila tenga su propio tamaño.

## Filas de diferente longitud

Para crear un array irregular no se especifica el tamaño de la segunda dimension en la instanciacion. Cada fila se crea individualmente con la longitud deseada.

```java
int[][] irregular = new int[3][]; // 3 filas, cada una sin definir aun

irregular[0] = new int[5]; // fila 0 con 5 elementos
irregular[1] = new int[3]; // fila 1 con 3 elementos
irregular[2] = new int[8]; // fila 2 con 8 elementos
```

La estructura resultante en memoria tiene la siguiente forma:

```
irregular -> [0] -> [0][1][2][3][4]
             [1] -> [0][1][2]
             [2] -> [0][1][2][3][4][5][6][7]
```

Tambien puede crearse directamente con initializer list:

```java
int[][] triangular = {
    {1},
    {2, 3},
    {4, 5, 6},
    {7, 8, 9, 10}
};
```

## Creacion con bucles

Los arrays irregulares se construyen frecuentemente con bucles cuando la estructura responde a una regla logica.

```java
// Triangulo de Floyd: fila i tiene i+1 elementos
int[][] floyd = new int[5][];
int contador = 1;

for (int i = 0; i < floyd.length; i++) {
    floyd[i] = new int[i + 1];
    for (int j = 0; j < floyd[i].length; j++) {
        floyd[i][j] = contador++;
    }
}

// Resultado:
// [1]
// [2, 3]
// [4, 5, 6]
// [7, 8, 9, 10]
// [11, 12, 13, 14, 15]
```

## Recorrido

El recorrido de arrays irregulares se hace con bucles anidados, pero la condicion del bucle interior debe usar `array[i].length` en lugar de una constante, porque cada fila tiene su propia longitud.

```java
int[][] triangular = {
    {1},
    {2, 3},
    {4, 5, 6},
    {7, 8, 9, 10}
};

for (int i = 0; i < triangular.length; i++) {
    for (int j = 0; j < triangular[i].length; j++) {
        System.out.print(triangular[i][j] + " ");
    }
    System.out.println();
}
```

Con for-each la sintaxis es identica a la de arrays rectangulares, ya que internamente se recorre cada fila como un array independiente:

```java
for (int[] fila : triangular) {
    for (int valor : fila) {
        System.out.print(valor + " ");
    }
    System.out.println();
}
```

## Uso practico

Los arrays irregulares aparecen en contextos donde los datos no tienen estructura rectangular natural.

**Almacenamiento de palabras por longitud:** cada fila contiene las palabras de una longitud determinada.

```java
String[][] palabrasPorLongitud = new String[10][];
// fila 2 -> palabras de 2 letras
// fila 5 -> palabras de 5 letras
```

**Matrices triangulares en algebra lineal:** las matrices triangulares superiores e inferiores tienen la mitad de los elementos que una matriz rectangular.

```java
// Matriz triangular superior 4x4
int[][] triangularSup = {
    {1, 2, 3, 4},
    {0, 5, 6, 7},
    {0, 0, 8, 9},
    {0, 0, 0, 10}
};
```

**Estructuras de datos en arbol:** cada nivel del arbol puede representarse como una fila con tantos elementos como nodos tenga ese nivel.

```java
// Representacion de un arbol binario por niveles
int[][] arbol = {
    {50},                // raiz
    {30, 70},            // nivel 2
    {20, 40, 60, 80}     // nivel 3
};
```

**Ahorro de memoria:** cuando los datos tienen distribucion irregular, un array jagged consume exactamente la memoria necesaria, sin desperdiciar espacio en posiciones no utilizadas como ocurriria con una matriz rectangular.

```java
// 1000 filas, cada fila con una longitud variable
int[][] ahorro = new int[1000][];
for (int i = 0; i < 1000; i++) {
    ahorro[i] = new int[calcularLongitud(i)];
}
```

> Un array rectangular de 1000x1000 ocupa 1,000,000 de posiciones aunque la mayoria no se usen. Un array jagged con las mismas filas pero longitudes variables puede reducir drasticamente el consumo de memoria si la dispersion de datos lo permite.
