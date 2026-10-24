# Arrays multidimensionales

Un array multidimensional en Java es un array cuyos elementos son a su vez arrays. Aunque se habla de arrays de dos, tres o mas dimensiones, en realidad se trata de arrays anidados: un array de arrays, donde cada nivel puede tener su propia longitud.

## Arrays 2D

El caso mas comun es el array bidimensional, que se utiliza para representar matrices, tablas y rejillas.

```java
// Declaracion e instanciacion de una matriz 3x4
int[][] matriz = new int[3][4];
```

Esto crea un array de 3 filas, donde cada fila es un array de 4 enteros. En memoria, `matriz` es un objeto que contiene 3 referencias a otros 3 objetos array (las filas).

```
matriz -> [ 0 ] [ 1 ] [ 2 ]
              |     |     |
              v     v     v
            [0][1] [0][1] [0][1]
            [2][3] [2][3] [2][3]
```

## Declaracion e inicializacion

Pueden declararse arrays de cualquier dimensionalidad anadiendo tantos pares de corchetes como dimensiones se necesiten.

```java
int[][] matriz;           // 2 dimensiones
int[][][] cubo;           // 3 dimensiones
int[][][][] hipercubo;    // 4 dimensiones
```

La inicializacion puede hacerse con `new` o con initializer list anidada:

```java
int[][] matriz = new int[3][4];

int[][] predefinida = {
    {1, 2, 3, 4},
    {5, 6, 7, 8},
    {9, 10, 11, 12}
};

String[][] nombres = {
    {"Juan", "Maria"},
    {"Carlos", "Ana", "Luis"},
    {"Sofia"}
};
```

## Acceso a elementos

Se accede a cada elemento mediante tantos indices como dimensiones tenga el array.

```java
int[][] matriz = {
    {1, 2, 3},
    {4, 5, 6},
    {7, 8, 9}
};

System.out.println(matriz[0][0]); // 1 (primera fila, primera columna)
System.out.println(matriz[1][2]); // 6 (segunda fila, tercera columna)

matriz[2][1] = 100; // modifica el elemento en fila 3, columna 2
```

El primer indice selecciona la fila (el array exterior). El segundo indice selecciona el elemento dentro del array de esa fila.

## Bucles anidados

Para recorrer un array bidimensional se necesitan dos bucles: uno exterior para las filas y uno interior para las columnas.

```java
int[][] matriz = {
    {1, 2, 3},
    {4, 5, 6},
    {7, 8, 9}
};

// Recorrido con for tradicional
for (int i = 0; i < matriz.length; i++) {
    for (int j = 0; j < matriz[i].length; j++) {
        System.out.print(matriz[i][j] + " ");
    }
    System.out.println();
}

// Recorrido con for-each
for (int[] fila : matriz) {
    for (int valor : fila) {
        System.out.print(valor + " ");
    }
    System.out.println();
}
```

`matriz.length` devuelve el numero de filas. `matriz[i].length` devuelve el numero de columnas de la fila i. Esto es util cuando las filas tienen longitudes diferentes (arrays irregulares).

## Arrays de mas dimensiones

Para tres dimensiones se anade un nivel mas de anidamiento. Un array tridimensional puede verse como un cubo de datos.

```java
int[][][] cubo = new int[3][4][5];
// 3 capas, cada capa con 4 filas, cada fila con 5 columnas

// Initializer list para 3D
int[][][] cubo = {
    {
        {1, 2},
        {3, 4}
    },
    {
        {5, 6},
        {7, 8}
    }
};

// Acceso y recorrido
for (int i = 0; i < cubo.length; i++) {
    for (int j = 0; j < cubo[i].length; j++) {
        for (int k = 0; k < cubo[i][j].length; k++) {
            System.out.println("cubo[" + i + "][" + j + "][" + k + "] = "
                + cubo[i][j][k]);
        }
    }
}
```

> Los arrays de 3 o mas dimensiones son poco frecuentes en aplicaciones reales. Cuando se necesitan, suelen modelarse mejor con objetos personalizados o colecciones anidadas. Un array 3D de 100x100x100 ya contiene un millon de elementos.

## La propiedad length en multidimensionales

`length` en cada nivel del array devuelve la longitud de ese nivel especifico.

```java
int[][][] cubo = new int[5][10][20];

System.out.println(cubo.length);       // 5  (capas)
System.out.println(cubo[0].length);    // 10 (filas por capa)
System.out.println(cubo[0][0].length); // 20 (columnas por fila)
```

Para arrays inicializados con literales, `length` se deduce automaticamente de los datos proporcionados.

```java
int[][] datos = {
    {1, 2, 3},
    {4, 5},
    {6}
};

System.out.println(datos.length);    // 3
System.out.println(datos[0].length); // 3
System.out.println(datos[1].length); // 2
System.out.println(datos[2].length); // 1
```
