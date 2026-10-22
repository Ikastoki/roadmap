<h1 align="center">Versiones</h1>

<h2>📑 Contenido</h2>

- [¿Que son las versiones?](#que-son-las-versiones)
- [Por Número](#por-número)
- [Fecha](#fecha)
- [Por Estabilidad](#por-estabilidad)

## ¿Que son las versiones?

En Git, las "versiones" se refieren a los diferentes estados o instantáneas del código y los archivos en un repositorio en diferentes momentos de su historia. Cada vez que realizas un "commit" en Git, estás creando una nueva versión del proyecto que representa un estado específico en el tiempo.

Cada versión, o commit, en Git está identificada por un hash SHA-1 único, que es una cadena larga de caracteres que actúa como una huella digital para ese commit en particular. Estos hashes aseguran que cada versión sea única y se pueda rastrear de manera confiable.

## Por Número

Algo común es realizar el manejo de versiones mediante 3 números: X.Y.Z y cada uno indica una cosa diferente:

- El primero (X) se le conoce como **versión mayor** y nos indica la versión principal del software. Ejemplo: 1.0.0, 3.0.0
- El segundo (Y) se le conoce como **versión menor** y nos indica nuevas funcionalidades. Ejemplo: 1.2.0, 3.3.0
- El tercero (Z) se le conoce como **revisión** y nos indica que se hizo una revisión del código por algún fallo. Ejemplo: 1.2.2, 3.3.4

**Parche**

En el caso de los parches podemos agregar un dígito para señalar el parche, ya teníamos algo así: X.Y.Z y ahora tendríamos algo así: X.Y.Z.P así que P sería el número del parche:

Ejemplo: 1.2.5.2, 02.03.03.01

## Fecha

En algunos necesitamos saber exactamente la fecha en que se publicó el software, entonces podremos utilizar el manejo de versiones por fecha. Este tiene muchas variaciones, se puede tener diferente orden del año, mes y día.

Ejemplo: 1.2.3.1543 donde 15 es el año 2015, 4 es el mes y 3 el día, como ya mencione anteriormente se podrían tener diferentes acomodos y formatos: 1.2.3.4315 o 1.2.3.201543, 1.2.3.1534

## Por Estabilidad

**Alpha** es una versión inestable que es muy probable que tenga muchas opciones que mejorar, pero queremos que sea probada para encontrar errores y poder poner a prueba funcionalidades, en la mayoría de los casos podemos decir que esta casi listo el producto. Ejemplo: 1.0Alpha, 1.0a1,1.0a2.

**Beta** una versión mas estable que Alpha en la que contamos con el producto en su totalidad, y se desea realizar pruebas de rendimiento, usabilidad y funcionamiento de algunos módulos para ver cómo funciona bajo un ambiente no tan controlado. Aquí aparece el nombre de Beta Tester que escuchamos mucho en el mundo del software. Ejemplo: 2.0Beta, 2.0b, 2.0b1


