# Anti-patrones en la arquitectura de capas

Un anti-patron es una solucion recurrente a un problema comun que, a diferencia de un patron genuino, genera mas problemas de los que resuelve. En el contexto de la arquitectura por capas, ciertos anti-patrones aparecen con frecuencia cuando el equipo no tiene suficiente disciplina arquitectonica o cuando la presion por entregar funcionalidades prima sobre la calidad estructural. Identificarlos a tiempo es la primera linea de defensa contra la degradacion del diseno.

## God Object (objeto omnipresente)

Un God Object (tambien llamado "clase todopoderosa") es una clase que acumula demasiada responsabilidad. En Java, un indicador heuristico ampliamente aceptado es que una clase supere las 300 lineas de codigo efectivo, sin contar imports ni metodos generados, aunque este umbral cambia segun la complejidad del dominio.

En la arquitectura por capas, el God Object suele aparecer como una clase de servicio que orquesta demasiadas operaciones o como un controlador que gestiona multiples endpoints no relacionados:

```java
// Sintoma de God Object
@Service
public class UsuarioService {
    // 20 metodos que abarcan:
    // - CRUD de usuarios
    // - Gestion de roles y permisos
    // - Envio de correos
    // - Generacion de reportes
    // - Logica de facturacion
}
```

Un God Object viola el principio de responsabilidad unica y produce varios problemas practicos: la clase es dificil de entender porque mezcla conceptos distintos; cualquier cambio requiere modificar la misma clase, aumentando el riesgo de regression; y el testing se vuelve costoso porque hay que instanciar muchas dependencias aunque solo se quiera probar un metodo.

La solucion es descomponer la clase en varias clases mas pequenas y cohesionadas, cada una con una responsabilidad definida. Si una clase de servicio tiene metodos que actuan sobre entidades diferentes (Usuario y Factura), esos metodos deberian estar en servicios distintos.

> Un God Object no se arregla anadiendo metodos, sino eliminando responsabilidades. La metrica util no es el numero de lineas, sino la cohesion: cuantos conceptos distintos maneja la clase.

## Anemic Domain Model (modelo de dominio anemico)

El Anemic Domain Model es un anti-patron en el que las entidades del dominio son meros contenedores de datos con getters y setters, sin comportamiento alguno. Toda la logica de negocio reside en los servicios, mientras que las entidades son estructuras pasivas.

```java
// Modelo anemico: solo datos
@Entity
public class Usuario {
    private Long id;
    private String email;
    private boolean activo;
    private int intentosFallidos;
    
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public String getEmail() { return email; }
    public void setEmail(String email) { this.email = email; }
    // mas getters y setters...
}

// Toda la logica esta aqui
@Service
public class UsuarioService {
    public void bloquearSiExcedeIntentos(Usuario usuario) {
        if (usuario.getIntentosFallidos() >= 3) {
            usuario.setActivo(false);
        }
    }
}
```

El problema fundamental es que el modelo de dominio debe encapsular comportamiento, no solo datos. La logica de `bloquearSiExcedeIntentos` pertenece a la entidad `Usuario`, no a un servicio externo. Cuando el comportamiento esta disperso en los servicios, cualquier cambio en una regla de negocio requiere buscar todas las apariciones de esa logica a traves del codigo, y las entidades se convierten en estructuras de datos intercambiables sin significado de dominio.

Para evitarlo, las entidades deben exponer metodos de negocio que operen sobre su estado interno, manteniendo los setters como ultimo recurso o eliminando los publicos por completo:

```java
@Entity
public class Usuario {
    private Long id;
    private String email;
    private boolean activo;
    private int intentosFallidos;
    
    public void registrarIntentoFallido() {
        this.intentosFallidos++;
        if (this.intentosFallidos >= 3) {
            this.activo = false;
        }
    }
    
    public boolean puedeIntentarLogin() {
        return this.activo;
    }
}
```

> El Anemic Domain Model suele ser senal de que el equipo piensa en terminos de tablas y registros en lugar de en terminos de objetos y comportamiento. No es un error de sintaxis, es un error de paradigma.

## Spaghetti Code (codigo sin estructura)

El Spaghetti Code describe un codigo cuya estructura de control es tan enrevesada que resulta imposible seguir el flujo de ejecucion. En la arquitectura por capas, esto se manifiesta cuando las dependencias entre capas no estan claras, cuando hay llamadas ciclicas entre servicios o cuando la logica de presentacion se mezcla con la de negocio y persistencia en un mismo metodo.

Un ejemplo tipico es un metodo de controlador que, ademas de manejar la peticion HTTP, realiza validaciones de negocio, construye consultas directamente contra la base de datos, itera sobre los resultados y aplica reglas de formato:

```java
@RestController
public class UsuarioController {
    @Autowired private JdbcTemplate jdbc;
    
    @GetMapping("/usuarios/activos")
    public List<Map<String, Object>> obtenerActivos() {
        // Logica de persistencia aqui (mal)
        List<Map<String, Object>> resultados = jdbc
            .queryForList("SELECT * FROM usuarios WHERE activo = 1");
        
        // Logica de negocio aqui (mal)
        List<Map<String, Object>> filtrados = new ArrayList<>();
        for (Map<String, Object> row : resultados) {
            if (((String)row.get("email")).endsWith("@empresa.com")) {
                row.put("tipo", "interno");
                filtrados.add(row);
            }
        }
        
        return filtrados;
    }
}
```

El Spaghetti Code se combate manteniendo la disciplina de capas: el controlador solo maneja la peticion y delega, el servicio contiene la logica de negocio y orquesta operaciones, el repositorio gestiona la persistencia. Cada capa tiene un proposito y no debe saltarse a la siguiente.

## Mixed Concerns (responsabilidades mezcladas)

Mixed Concerns es la variante mas sutil del Spaghetti Code: ocurre cuando una clase o metodo maneja dos o mas responsabilidades distintas sin que ninguna domine claramente. A diferencia del God Object, que es visible por su tamano, Mixed Concerns puede aparecer en clases pequenas que, sin embargo, mezclan conceptos ortogonales.

Ejemplos comunes: un metodo que valida una entidad y ademas la persiste; un metodo que calcula un valor y ademas formatea la salida; una clase que maneja la autenticacion y ademas el envio de notificaciones.

```java
// Mixed concerns: validacion + persistencia + logging en el mismo metodo
@Service
public class PedidoService {
    public PedidoDTO crearPedido(CrearPedidoRequest request) {
        // Validacion (responsabilidad de la capa de validacion)
        if (request.getItems().isEmpty()) {
            throw new IllegalArgumentException("El pedido debe tener items");
        }
        
        // Logica de negocio
        Pedido pedido = new Pedido(request.getClienteId());
        // ...
        
        // Logging (responsabilidad transversal)
        logger.info("Pedido creado: " + pedido.getId());
        
        // Persistencia
        pedidoRepository.save(pedido);
        
        return PedidoMapper.toDTO(pedido);
    }
}
```

La solucion pasa por separar las responsabilidades en diferentes componentes: la validacion se realiza antes de entrar al servicio (o mediante Bean Validation), el logging se implementa con aspectos AOP (como `@Loggable`), y el servicio se concentra exclusivamente en la logica de negocio que orquesta.

> La diferencia entre el diseno y el caos no es la complejidad del dominio, sino la claridad con la que cada componente asume una sola responsabilidad y no invade la de los demas.
