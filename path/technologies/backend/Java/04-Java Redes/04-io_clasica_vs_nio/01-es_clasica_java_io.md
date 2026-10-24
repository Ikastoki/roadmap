# E/S clasica (java.io)

El paquete `java.io` constituye la capa de entrada/salida original de Java, presente desde la version 1.0. Su diseno se basa en el patron decorador y en el concepto de flujo (stream) como secuencia ordenada de bytes o caracteres que fluye desde un origen hacia un destino. La abstraccion central es que un programa lee datos de un stream de entrada y escribe datos hacia un stream de salida, sin que exista un concepto equivalente al de canal bidireccional que apareceria anos despues con NIO.

## Flujos de bytes: InputStream y OutputStream

`InputStream` y `OutputStream` son clases abstractas que definen el contrato basico para la lectura y escritura de bytes. `InputStream` declara el metodo `int read()` que lee el siguiente byte y lo devuelve como un entero entre 0 y 255, o -1 si se ha alcanzado el final del flujo. `OutputStream` declara `void write(int b)` que escribe el byte menos significativo del entero. Sobre estos cimientos se construyen flujos especializados: `FileInputStream` lee desde un archivo del sistema de archivos, `ByteArrayInputStream` lee desde un arreglo de bytes en memoria, y `FilterInputStream` actua como clase base para decoradores como `BufferedInputStream` y `DataInputStream`.

```java
InputStream fis = new FileInputStream("datos.bin");
int byteLeido;
while ((byteLeido = fis.read()) != -1) {
    // Procesar byte
}
fis.close();
```

El uso de `read()` sin buffer implica una llamada al sistema operativo por cada byte, lo que resulta ineficiente. Por eso se envuelve el flujo en un `BufferedInputStream`, que agrega un buffer interno de 8192 bytes por defecto y reduce drasticamente las llamadas nativas.

```java
InputStream bis = new BufferedInputStream(new FileInputStream("datos.bin"));
```

## Flujos de caracteres: Reader y Writer

Java separa los flujos de bytes de los flujos de caracteres para manejar correctamente la codificacion de texto. `Reader` y `Writer` son las clases abstractas que definen la lectura y escritura de caracteres Unicode. `FileReader` y `FileWriter` proporcionan acceso directo a archivos de texto, aunque con una limitacion importante: utilizan la codificacion por defecto de la plataforma, lo que hace que su comportamiento no sea portable. `InputStreamReader` actua como puente entre un `InputStream` (bytes) y un `Reader` (caracteres), aplicando una codificacion explicita.

```java
Reader reader = new InputStreamReader(
    new FileInputStream("datos.txt"), StandardCharsets.UTF_8);
```

`BufferedReader` anade un buffer y, mas importante, el metodo `readLine()` que permite leer lineas completas de texto sin tener que gestionar la delimitacion manualmente. Este metodo es probablemente la operacion de E/S de caracteres mas utilizada en Java clasico.

```java
BufferedReader br = new BufferedReader(new FileReader("texto.txt"));
String linea;
while ((linea = br.readLine()) != null) {
    System.out.println(linea);
}
br.close();
```

## DataInputStream y DataOutputStream

Para leer y escribir tipos primitivos de Java (int, long, float, double) de forma portatil desde un stream binario, se utilizan `DataInputStream` y `DataOutputStream`. Estos decoradores permiten leer datos en formato binario con independencia de la plataforma, utilizando el orden de bytes de red (big-endian) por defecto. `DataInputStream.readInt()` lee cuatro bytes y los interpreta como un entero de 32 bits, mientras que `DataOutputStream.writeDouble()` escribe un double de 64 bits en formato IEEE 754.

```java
DataInputStream dis = new DataInputStream(
    new BufferedInputStream(new FileInputStream("datos.bin")));
int entero = dis.readInt();
long largo = dis.readLong();
double decimal = dis.readDouble();
String texto = dis.readUTF(); // String codificado en UTF-8 modificado
dis.close();
```

`readUTF()` utiliza una variante de UTF-8 donde el tamano del string se almacena como un entero de 16 bits sin signo al inicio, limitando la longitud maxima a 65535 caracteres. Para cadenas mas largas debe recurrirse a otros mecanismos.

## ObjectInputStream y ObjectOutputStream

La serializacion de objetos en Java permite convertir un grafo de objetos en una secuencia de bytes que puede ser persistida o transmitida por la red. `ObjectOutputStream` serializa objetos implementando la interfaz `Serializable`, y `ObjectInputStream` los reconstruye. El mecanismo utiliza reflexion para recorrer los campos del objeto y maneja automaticamente las referencias circulares y las duplicadas.

```java
ObjectOutputStream oos = new ObjectOutputStream(
    new FileOutputStream("objeto.ser"));
oos.writeObject(new Usuario("juan", "juan@example.com"));
oos.close();

ObjectInputStream ois = new ObjectInputStream(
    new FileInputStream("objeto.ser"));
Usuario usuario = (Usuario) ois.readObject();
ois.close();
```

La serializacion introduce problemas de versionado entre emisor y receptor: si la clase cambia (se anade o elimina un campo), la deserializacion puede fallar. Para controlar esto se utiliza `serialVersionUID`, un identificador unico de version que debe declararse explicitamente en cada clase serializable.

## Limitaciones del modelo clasico

El diseno de `java.io` presenta tres limitaciones fundamentales que motivaron la creacion de NIO:

**Bloqueante.** Cada operacion de lectura o escritura bloquea el hilo de ejecucion hasta que el dato este disponible o el dispositivo complete la operacion. Si un `read()` se invoca sobre un socket y no hay datos, el hilo queda suspendido hasta que lleguen. Esto impide que un mismo hilo atienda multiples conexiones simultaneas.

**Un hilo por operacion.** La unica forma de manejar multiples conexiones de red con E/S clasica es asignar un hilo independiente por cada conexion. Cada hilo consume memoria de pila (tipicamente 1 MB por hilo en la JVM) y recursos del planificador del sistema operativo. Con miles de conexiones simultaneas, el costo en memoria y cambio de contexto se vuelve prohibitivo.

**No escalable.** El modelo de un hilo por conexion no escala horizontalmente. Para 10.000 conexiones simultaneas se necesitarian 10.000 hilos, lo que excede la capacidad de cualquier maquina virtual convencional. Surge entonces el denominado problema C10K (10.000 conexiones simultaneas), que NIO aborda con un diseno no bloqueante y multiplexado.

> El modelo clasico de `java.io` no es inherentemente incorrecto; es simplemente inadecuado para escenarios de alta concurrencia en E/S de red. Para operaciones de archivo locales con volumen moderado o para aplicaciones con pocas conexiones simultaneas, sigue siendo perfectamente valido y, de hecho, mas simple de implementar.
