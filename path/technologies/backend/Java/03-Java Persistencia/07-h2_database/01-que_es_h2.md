# Que es H2

H2 es un sistema de gestion de bases de datos relacional escrito completamente en Java. Su principal caracteristica distintiva es la capacidad de ejecutarse en modo embebido, es decir, dentro del mismo proceso de la aplicacion sin necesidad de instalar un servidor de base de datos independiente. Esto lo convierte en una herramienta fundamental para desarrolladores Java que necesitan entornos de prueba rapidos, entornos de integracion continua o incluso aplicaciones de escritorio que requieren persistencia local sin dependencias externas.

El motor soporta un subconjunto muy amplio del estandar SQL, incluyendo joins, subconsultas, funciones de ventana, tablas temporales, indices, secuencias y restricciones de integridad referencial. H2 es compatible con modos de compatibilidad con MySQL y PostgreSQL, lo que permite emular el comportamiento dialectico de estas bases de datos durante el desarrollo.

## Modo embebido

En modo embebido, H2 se ejecuta en la misma maquina virtual que la aplicacion. No se abre ningun puerto de red ni se requiere un proceso separado. La conexion se realiza directamente a traves del classpath, lo que elimina la latencia de red y ofrece el maximo rendimiento posible. Este modo es el mas utilizado en entornos de testing y desarrollo local.

```java
// Conexion en modo embebido con base de datos en memoria
Connection conn = DriverManager.getConnection("jdbc:h2:mem:testdb");

// Conexion en modo embebido con persistencia en disco
Connection conn = DriverManager.getConnection("jdbc:h2:~/datos/mibd");
```

La base de datos se crea automaticamente si no existe, tanto en memoria como en disco. En modo memoria, los datos se pierden al cerrar la conexion o detener la JVM. En modo disco, H2 escribe los datos en un archivo binario y un archivo de traza.

## Modo servidor

H2 tambien puede ejecutarse como un servidor TCP independiente, permitiendo que multiples aplicaciones o procesos accedan a la misma base de datos simultaneamente. Este modo es util cuando se require compartir el estado de la base de datos entre varios servicios o cuando se necesita acceder desde herramientas externas como DBeaver o IntelliJ.

```bash
# Iniciar el servidor H2 desde la linea de comandos
java -cp h2*.jar org.h2.tools.Server -tcp -tcpAllowOthers -web -webAllowOthers
```

Una vez iniciado el servidor, las aplicaciones se conectan mediante la URL `jdbc:h2:tcp://localhost:9092/~/<dbname>`. El parametro `-web` activa la consola web en el puerto 8082.

## Compatibilidad con otros motores

H2 ofrece modos de compatibilidad que permiten emular el comportamiento de MySQL y PostgreSQL. Aunque no son replicas exactas, estos modos ajustan el dialecto SQL y algunas funciones especificas.

| Modo | URL | Comportamiento |
|------|-----|---------------|
| MySQL | `jdbc:h2:mem:test;MODE=MySQL` | Funciones de fecha, comillas invertidas, auto_increment |
| PostgreSQL | `jdbc:h2:mem:test;MODE=PostgreSQL` | Secuencias, schemas public, function CURRENT_TIMESTAMP |

> El modo de compatibilidad no garantiza una equivalencia perfecta. Existen diferencias en la gestion de tipos de datos, funciones nativas y comportamiento transaccional. Para entornos de produccion con PostgreSQL, se recomienda utilizar TestContainers o una instancia real.

## H2 Console web

H2 incluye una consola web integrada que permite explorar la base de datos, ejecutar consultas SQL y administrar tablas sin necesidad de herramientas externas. En modo servidor, esta consola esta disponible en `http://localhost:8082/`. En modo embebido dentro de una aplicacion Spring Boot, se accede en `http://localhost:8080/h2-console`.

```properties
# Habilitar la consola H2 en Spring Boot
spring.h2.console.enabled=true
spring.h2.console.path=/h2-console
```

La consola es una aplicacion web liviana escrita con un frontend basado en JavaScript vanilla. Permite autocompletado de sentencias SQL, visualizacion del plan de ejecucion y exportacion de resultados en formato CSV.

## Casos de uso tipicos

H2 es la opcion preferida para:
- Pruebas unitarias y de integracion que requieren una base de datos real sin levantar infraestructura.
- Prototipado rapido y desarrollo local donde la persistencia en disco es suficiente.
- Aplicaciones de escritorio Java que necesitan una base de datos local autocontenida.
- Cursos y entornos de formacion donde los alumnos no deben instalar servidores de bases de datos.

Para entornos de produccion reales, H2 no es recomendable. Carece de caracteristicas como replicacion, clustering, alta disponibilidad o rendimiento optimizado para altas cargas de trabajo concurrentes. Su lugar esta en el ciclo de desarrollo y testing, no en produccion.
