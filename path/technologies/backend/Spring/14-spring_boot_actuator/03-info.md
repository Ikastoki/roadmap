# Info

El endpoint `/actuator/info` expone metadatos generales sobre la aplicacion que son utiles para identificar la version, el entorno y el contexto de ejecucion. A diferencia de otros endpoints de Actuator que reportan datos operativos, `/info` proporciona informacion estatica configurada por el desarrollador o generada automaticamente por la aplicacion.

## Configuracion basica

Los datos de info se definen en el archivo de configuracion bajo el prefijo `info` y se exponen directamente sin necesidad de codigo adicional.

```yaml
info:
  app:
    nombre: Mi Aplicacion
    version: 2.1.0
    descripcion: Sistema de gestion de pedidos
    autor: Equipo de Desarrollo
  contacto:
    email: soporte@ejemplo.com
    url: https://ejemplo.com/soporte
  licencia:
    nombre: Apache 2.0
    url: https://www.apache.org/licenses/LICENSE-2.0
```

```bash
curl http://localhost:8080/actuator/info
```

```json
{
  "app": {
    "nombre": "Mi Aplicacion",
    "version": "2.1.0",
    "descripcion": "Sistema de gestion de pedidos",
    "autor": "Equipo de Desarrollo"
  },
  "contacto": {
    "email": "soporte@ejemplo.com",
    "url": "https://ejemplo.com/soporte"
  },
  "licencia": {
    "nombre": "Apache 2.0",
    "url": "https://www.apache.org/licenses/LICENSE-2.0"
  }
}
```

## Build info

El plugin de Spring Boot puede generar automaticamente un archivo `META-INF/build-info.properties` con datos del build que se incluyen en el endpoint `/info`. Esto permite identificar exactamente que version del codigo esta desplegada en cada entorno.

```xml
<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
    <executions>
        <execution>
            <goals>
                <goal>build-info</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

```yaml
management:
  info:
    build:
      enabled: true
      mode: full
```

```json
{
  "build": {
    "artifact": "mi-aplicacion",
    "name": "mi-aplicacion",
    "version": "2.1.0-SNAPSHOT",
    "time": "2026-07-25T10:30:00Z",
    "group": "com.ejemplo"
  }
}
```

## Git info

Spring Boot puede generar informacion del repositorio Git a partir del plugin `git-commit-id-plugin` (Maven) o `org.ajoberstar.grgit` (Gradle). Esta informacion incluye el SHA del commit, la branch y el estado del repositorio en el momento del build.

```xml
<plugin>
    <groupId>pl.project13.maven</groupId>
    <artifactId>git-commit-id-plugin</artifactId>
    <executions>
        <execution>
            <goals>
                <goal>revision</goal>
            </goals>
            <configuration>
                <failOnNoGitDirectory>false</failOnNoGitDirectory>
            </configuration>
        </execution>
    </executions>
</plugin>
```

```yaml
management:
  info:
    git:
      enabled: true
      mode: full
      branch: main
```

```json
{
  "git": {
    "branch": "main",
    "commit": {
      "id": "a1b2c3d",
      "time": "2026-07-25T09:15:00Z",
      "message": {
        "full": "feat: agregar endpoint de pedidos"
      }
    },
    "dirty": false
  }
}
```

> En entornos de produccion, exponer el SHA del commit y la branch permite correlacionar rapidamente un comportamiento inesperado con un cambio especifico en el codigo fuente.

## Custom InfoContributor

`InfoContributor` es la interfaz que permite contribuir datos personalizados al endpoint `/info`. Spring Boot proporciona implementaciones para build y git, pero se pueden crear contribuciones propias implementando esta interfaz.

```java
@Component
public class BaseDatosInfoContributor implements InfoContributor {

    private final DataSource dataSource;

    public BaseDatosInfoContributor(DataSource dataSource) {
        this.dataSource = dataSource;
    }

    @Override
    public void contribute(Info.Builder builder) {
        try (Connection conn = dataSource.getConnection()) {
            DatabaseMetaData meta = conn.getMetaData();
            builder.withDetail("database", Map.of(
                "producto", meta.getDatabaseProductName(),
                "version", meta.getDatabaseProductVersion(),
                "url", meta.getURL()
            ));
        } catch (SQLException e) {
            builder.withDetail("database", Map.of(
                "error", "No se pudo obtener informacion de la BD"
            ));
        }
    }
}
```

```java
@Component
public class EntornoInfoContributor implements InfoContributor {

    @Value("${app.entorno:local}")
    private String entorno;

    @Value("${app.region:eu-west-1}")
    private String region;

    @Override
    public void contribute(Info.Builder builder) {
        builder.withDetail("entorno", Map.of(
            "nombre", entorno,
            "region", region,
            "java.version", System.getProperty("java.version"),
            "uptime", ManagementFactory.getRuntimeMXBean().getUptime() / 1000 + "s"
        ));
    }
}
```

La contribucion de informacion a traves de `InfoContributor` permite centralizar la logica de obtencion de metadatos y exponerla de forma consistente en todos los entornos. Cada contribuidor se ejecuta de forma independiente, por lo que un error en uno no afecta a los demas.

