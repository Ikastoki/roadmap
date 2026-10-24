# Responsabilidades por capa

Cada capa de la arquitectura define un contrato explicito de lo que debe y no debe hacer. Respetar estos limites es lo que convierte un conjunto de clases en una arquitectura en capas funcional. Cuando una capa incumple su responsabilidad --un controlador que ejecuta SQL, un servicio que genera JSON, un repositorio que implementa reglas de negocio-- el sistema pierde las ventajas de la separacion: el cambio local se vuelve global y las pruebas se complican innecesariamente.

## Responsabilidades de la capa de presentacion

La capa de presentacion actua como frontera del sistema. Su responsabilidad primaria es traducir las peticiones del mundo exterior al lenguaje interno de la aplicacion y viceversa.

**Responsabilidades:**

- Recibir y parsear peticiones HTTP, mensajes de cola o eventos de entrada.
- Validar la estructura y el formato de los datos entrantes (validacion sintactica).
- Invocar la capa de negocio pasando los datos ya validados.
- Capturar excepciones de la capa de negocio y traducirlas a codigos de estado HTTP adecuados (404 para "no encontrado", 400 para "solicitud invalida", 403 para "no autorizado").
- Serializar la respuesta al formato acordado con el cliente (JSON, XML, HTML).

**No debe hacer:**

- Aplicar reglas de negocio, como calcular descuentos o verificar permisos.
- Acceder directamente a repositorios o bases de datos.
- Mantener estado entre peticiones (salvo el estado de sesion gestionado por el contenedor web).
- Decidir que flujo de ejecucion sigue el sistema segun condiciones de negocio.

```java
// MAL: el controlador aplica una regla de negocio
@PostMapping("/transferir")
public ResponseEntity<?> transferir(@RequestBody TransferenciaRequest req) {
    if (req.getImporte() > 10000) {
        return ResponseEntity.status(403).body("Requiere aprobacion adicional");
    }
    // ...
}

// BIEN: el controlador delega en el servicio
@PostMapping("/transferir")
public ResponseEntity<?> transferir(@RequestBody TransferenciaRequest req) {
    ResultadoTransferencia resultado = service.ejecutar(req);
    return ResponseEntity.status(resultado.esExito() ? 200 : 422).body(resultado);
}
```

## Responsabilidades de la capa de negocio

La capa de negocio es el corazon de la aplicacion. Alberga las reglas que justifican la existencia del sistema y orquesta las operaciones entre los distintos repositorios y servicios auxiliares.

**Responsabilidades:**

- Implementar los casos de uso y las reglas de negocio de la aplicacion.
- Orquestar llamadas a repositorios y servicios externos siguiendo un flujo definido.
- Gestionar transacciones que abarquen multiples operaciones de persistencia.
- Validar semanticamente los datos (reglas de dominio, no formato).
- Lanzar excepciones de negocio cuando una regla se incumple (saldo insuficiente, duplicado no permitido).

**No debe hacer:**

- Recibir peticiones HTTP ni serializar respuestas.
- Conocer la tecnologia de persistencia subyacente (SQL, JPA, MongoDB).
- Mantener estado conversacional propio de la sesion del usuario.
- Depender de anotaciones o imports de frameworks de presentacion (javax.servlet, org.springframework.web.bind.annotation).

> La inyeccion de dependencias es el mecanismo que mantiene pura la capa de negocio. El servicio declara sus dependencias como interfaces; el contenedor de IoC se encarga de suministrar las implementaciones concretas. Asi la capa de negocio nunca importa clases concretas de persistencia o presentacion.

## Responsabilidades de la capa de acceso a datos

La capa de acceso a datos aisla los detalles de persistencia del resto de la aplicacion. Su existencia permite cambiar de base de datos o de estrategia de almacenamiento sin afectar a los servicios de negocio.

**Responsabilidades:**

- Proveer interfaces de repositorio que la capa de negocio pueda consumir.
- Implementar las operaciones CRUD y consultas especificas contra el medio de almacenamiento.
- Mapear entre el esquema relacional (tablas, columnas) y los objetos de dominio.
- Gestionar la conexion con la base de datos y el ciclo de vida de los recursos (conexiones, statements, resultsets).
- Ejecutar las migraciones de esquema al iniciar la aplicacion.

**No debe hacer:**

- Implementar reglas de negocio ni validaciones semanticas.
- Decidir aspectos de presentacion o formato de salida.
- Lanzar excepciones propias del driver de base de datos hacia las capas superiores (debe traducirlas a excepciones de la aplicacion).
- Realizar operaciones de logica transaccional que abarquen multiples repositorios.

```java
// MAL: el repositorio decide una regla de negocio
public void guardar(Usuario usuario) {
    if (usuario.getSaldo() < 0) {
        throw new IllegalStateException("El saldo no puede ser negativo");
    }
    jdbc.update("INSERT INTO usuarios ...");
}

// BIEN: el repositorio solo persiste
public void guardar(Usuario usuario) {
    jdbc.update("INSERT INTO usuarios (nombre, email, saldo) VALUES (?, ?, ?)",
        usuario.getNombre(), usuario.getEmail(), usuario.getSaldo());
}
```

## Responsabilidades de la base de datos

La base de datos es el almacen fisico y la garantia de integridad del sistema. Su responsabilidad abarca la gestion del ciclo de vida de los datos y la eficiencia de las operaciones de consulta.

**Responsabilidades:**

- Almacenar los datos de forma persistente y duradera.
- Garantizar la integridad referencial mediante claves foraneas y constraints.
- Proveer indices que optimicen el tiempo de ejecucion de las consultas.
- Asegurar la atomicidad, consistencia, aislamiento y durabilidad (ACID) de las transacciones.
- Ejecutar migraciones de esquema que evolucionen la estructura de datos.

**No debe hacer:**

- Contener logica de negocio de la aplicacion en procedimientos almacenados o triggers (salvo casos excepcionales justificados por rendimiento).
- Gestionar la serializacion o el formateo de datos para presentacion.
- Implementar flujos de autorizacion que deberian estar en la capa de negocio.

> Las migraciones son el equivalente al control de versiones para la base de datos. Sin migraciones, los cambios de esquema se aplican manualmente en cada entorno, generando desviaciones imposibles de rastrear. Flyway y Liquibase son las herramientas mas extendidas en el ecosistema Java para este proposito.

## Resumen de responsabilidades

| Capa | Que hace | Que no hace |
|---|---|---|
| Presentacion | Valida entrada, invoca servicios, formatea respuesta | Reglas de negocio, acceso a datos |
| Negocio | Reglas de dominio, orquestacion, transacciones | HTTP, SQL, serializacion |
| Acceso a datos | CRUD, consultas, mapeo | Reglas de negocio, presentacion |
| Base de datos | Almacenamiento, integridad, indices | Logica de aplicacion, formateo |

La tabla anterior no es un dogma, sino un contrato de diseno. En proyectos reales aparecen zonas grises (una validacion de unicidad suele implementarse tanto en negocio como con un constraint UNIQUE en base de datos), pero la direccion general debe ser siempre mantener cada responsabilidad en su capa correspondiente para preservar la mantenibilidad del sistema.
