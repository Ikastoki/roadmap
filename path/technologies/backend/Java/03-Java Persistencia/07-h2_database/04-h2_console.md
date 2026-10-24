# H2 Console

H2 incluye una consola web integrada que permite interactuar con la base de datos a traves de una interfaz grafica en el navegador. Esta herramienta es util durante el desarrollo para inspeccionar tablas, ejecutar consultas ad-hoc y verificar el estado de los datos sin necesidad de herramientas externas como DBeaver o DataGrip.

## Acceso a la consola

Cuando H2 se ejecuta en modo servidor independiente, la consola web esta disponible en `http://localhost:8082/`. Cuando se integra dentro de una aplicacion Spring Boot, la URL por defecto es `http://localhost:8080/h2-console`.

```properties
# Habilitar consola en Spring Boot
spring.h2.console.enabled=true
spring.h2.console.path=/h2-console
```

La consola solo debe activarse en perfiles de desarrollo. En produccion debe permanecer deshabilitada por razones de seguridad.

```properties
# application-dev.properties
spring.h2.console.enabled=true

# application-prod.properties
spring.h2.console.enabled=false
```

## Pantalla de inicio de sesion

Al acceder a la consola, se presenta un formulario con los siguientes campos:
- **Saved Settings**: perfiles de conexion guardados localmente en el navegador.
- **Setting Name**: nombre para guardar la configuracion actual.
- **Driver Class**: clase del driver JDBC. Para H2 es `org.h2.Driver`.
- **JDBC URL**: URL de conexion a la base de datos.
- **User Name**: nombre de usuario (por defecto `sa`).
- **Password**: contrasena (por defecto vacia).

Para una aplicacion Spring Boot con H2 en memoria, la configuracion tipica es:
- Driver Class: `org.h2.Driver`
- JDBC URL: `jdbc:h2:mem:testdb`
- User Name: `sa`
- Password: (vacio)

## Exploracion del esquema

Una vez conectados, el panel izquierdo muestra el arbol del esquema de la base de datos. Desde ahi se puede:
- Listar tablas, vistas, secuencias e indices.
- Expandir cada tabla para ver sus columnas, tipos, restricciones y claves foraneas.
- Navegar entre esquemas (`PUBLIC` por defecto en H2).

Al hacer clic en una tabla, el panel derecho genera automaticamente una consulta `SELECT * FROM tabla LIMIT 100` y muestra los resultados.

## Ejecucion de SQL

El panel derecho contiene un editor SQL con las siguientes capacidades:
- Resaltado de sintaxis para palabras clave SQL.
- Autocompletado de nombres de tablas, columnas y funciones.
- Ejecucion de sentencias con el boton Run o el atajo Ctrl+Enter.
- Ejecucion de multiples sentencias separadas por punto y coma.
- Visualizacion del plan de ejecucion con `EXPLAIN SELECT ...`.

```sql
-- Consulta de ejemplo en la consola
SELECT t.table_name,
       c.column_name,
       c.data_type,
       c.is_nullable
FROM information_schema.tables t
JOIN information_schema.columns c
  ON t.table_name = c.table_name
WHERE t.table_schema = 'PUBLIC'
ORDER BY t.table_name, c.ordinal_position;
```

Los resultados se muestran en una tabla paginada. Se puede hacer clic en cualquier valor para editarlo directamente si la consulta es sobre una unica tabla con clave primaria.

## Configuracion avanzada de la consola

Spring Boot expone varias propiedades para personalizar el comportamiento de la consola:

```properties
spring.h2.console.enabled=true
spring.h2.console.path=/custom-path
spring.h2.console.settings.web-allow-others=true
spring.h2.console.settings.trace=false
```

- `web-allow-others`: permite conexiones desde otras maquinas (por defecto solo localhost).
- `trace`: activa el registro detallado de las operaciones realizadas desde la consola.

> Habilitar `web-allow-others` en entornos que no sean de desarrollo supone un riesgo de seguridad. Cualquier persona con acceso a la red podria conectarse a la base de datos si conoce la URL y credenciales.

## Limitaciones de la consola

La consola H2 es una herramienta funcional pero tiene limitaciones importantes:
- No soporta multiples pestañas de consulta simultaneas.
- El editor SQL carece de formato automatico de codigo.
- La visualizacion de planes de ejecucion es basica comparada con herramientas como pgAdmin.
- No ofrece herramientas de profiling o analisis de rendimiento.

Para trabajo intensivo con bases de datos, es preferible usar DBeaver, DataGrip o el cliente nativo del motor correspondiente. La consola H2 es adecuada para consultas rapidas durante el desarrollo, pero no para tareas de administracion complejas.

## Seguridad

La consola H2 no debe estar accesible en produccion. Ademas de deshabilitarla con `spring.h2.console.enabled=false`, conviene asegurarse de que la dependencia de H2 tenga scope `test` o `runtime` en el gestor de dependencias, evitando que este disponible en el classpath de produccion.

```xml
<dependency>
    <groupId>com.h2database</groupId>
    <artifactId>h2</artifactId>
    <scope>test</scope>
</dependency>
```
