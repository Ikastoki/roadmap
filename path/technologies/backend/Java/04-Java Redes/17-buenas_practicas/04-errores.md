# Manejo de errores

Las aplicaciones de red operan en un entorno hostil: la red falla, los servidores remotos se caen, los timeouts se disparan y los datos llegan corruptos o incompletos. Un manejo de errores robusto no solo recupera la aplicacion de fallos transitorios sino que evita que un fallo en un componente secundario degrade o derribe todo el sistema.

## IOException y sus subtipos

`IOException` es la excepcion raiz de la mayoria de errores de red en Java. Es una excepcion comprobada (checked exception) que debe manejarse explicitamente.

```java
try (Socket socket = new Socket("ejemplo.com", 80)) {
    // operaciones...
} catch (UnknownHostException e) {
    log.error("Host desconocido: {}", e.getMessage());
    // DNS fallo, no reintentar
} catch (ConnectException e) {
    log.error("Conexion rechazada: {}", e.getMessage());
    // Servidor caido o firewall bloqueando
} catch (SocketTimeoutException e) {
    log.warn("Timeout de conexion: {}", e.getMessage());
    // Reintentar con backoff
} catch (IOException e) {
    log.error("Error de red: {}", e.getMessage());
    // Error generico
}
```

Clasificar el error permite decidir la accion correcta: un `UnknownHostException` no tiene sentido reintentarlo (el DNS no va a cambiar solo), mientras que un `SocketTimeoutException` puede resolverse con un reintento.

## Reintentos con backoff exponencial

Un reintento inmediato tras un fallo de red suele fallar de nuevo porque el problema persiste. El backoff exponencial incrementa el tiempo de espera entre reintentos, dando tiempo al sistema a recuperarse.

```java
import java.time.Duration;
import java.util.function.Supplier;

public class RetryUtil {
    private static final int MAX_INTENTOS = 3;

    public static <T> T conReintentos(
            Supplier<T> operacion,
            String nombreOperacion) {
        long espera = 1000; // 1 segundo inicial
        for (int intento = 1; intento <= MAX_INTENTOS; intento++) {
            try {
                return operacion.get();
            } catch (Exception e) {
                if (intento == MAX_INTENTOS) {
                    throw new RuntimeException(
                            "Fallo tras " + MAX_INTENTOS
                                    + " intentos: "
                                    + nombreOperacion, e);
                }
                log.warn("Intento {} fallido para {}. Reintentando en {}ms",
                        intento, nombreOperacion, espera);
                try {
                    Thread.sleep(espera);
                } catch (InterruptedException ie) {
                    Thread.currentThread().interrupt();
                    throw new RuntimeException(
                            "Reintento interrumpido", ie);
                }
                espera *= 2; // backoff exponencial
                // Anadir jitter: espera += random(0, 1000)
            }
        }
        throw new IllegalStateException(
                "No deberia llegar aqui");
    }
}
```

Uso:

```java
String respuesta = RetryUtil.conReintentos(() -> {
    try {
        return cliente.send(peticion,
                HttpResponse.BodyHandlers.ofString()).body();
    } catch (IOException e) {
        throw new RuntimeException(e);
    }
}, "GET /api/datos");
```

> El jitter (componente aleatorio anadido al tiempo de espera) evita el efecto thundering herd, donde todos los clientes reintentan simultaneamente tras una caida y saturan el servidor.

## Circuit breaker

El patron circuit breaker previene que llamadas repetidas a un servicio fallido agoten los recursos del sistema. Cuando el numero de fallos supera un umbral, el circuito se abre y las llamadas fallan inmediatamente sin intentar la conexion real.

```java
public class CircuitBreaker {
    private enum Estado { CERRADO, ABIERTO, SEMI_ABIERTO }

    private Estado estado = Estado.CERRADO;
    private int fallosConsecutivos = 0;
    private final int umbralFallos;
    private final long timeoutAperturaMs;
    private long ultimoFallo;

    public CircuitBreaker(int umbralFallos, long timeoutAperturaMs) {
        this.umbralFallos = umbralFallos;
        this.timeoutAperturaMs = timeoutAperturaMs;
    }

    public synchronized <T> T llamar(Supplier<T> operacion) {
        if (estado == Estado.ABIERTO) {
            if (System.currentTimeMillis() - ultimoFallo
                    < timeoutAperturaMs) {
                throw new RuntimeException(
                        "Circuito abierto, llamada rechazada");
            }
            estado = Estado.SEMI_ABIERTO;
        }

        try {
            T resultado = operacion.get();
            if (estado == Estado.SEMI_ABIERTO) {
                estado = Estado.CERRADO;
                fallosConsecutivos = 0;
            }
            return resultado;
        } catch (Exception e) {
            fallosConsecutivos++;
            ultimoFallo = System.currentTimeMillis();
            if (fallosConsecutivos >= umbralFallos) {
                estado = Estado.ABIERTO;
            }
            throw e;
        }
    }
}
```

```
CERRADO -> fallos superan umbral -> ABIERTO
ABIERTO -> timeout alcanzado -> SEMI_ABIERTO
SEMI_ABIERTO -> exito -> CERRADO
SEMI_ABIERTO -> fallo -> ABIERTO (reinciar timeout)
```

> No implementar circuit breakers manualmente en produccion. Resilience4j y Spring Cloud Circuit Breaker son bibliotecas maduras que manejan estados, hilos y metricas correctamente.

## Degradacion elegante (graceful degradation)

Cuando un servicio dependiente falla, la aplicacion no debe fallar en cascada. Debe degradar su funcionalidad de forma controlada.

```java
public class ServicioUsuarios {
    private final ServicioRemoto remoto;
    private final CacheLocal<String, Usuario> cache;

    public Usuario obtenerUsuario(Long id) {
        try {
            Usuario u = remoto.buscarPorId(id);
            cache.put("usr-" + id, u);
            return u;
        } catch (Exception e) {
            log.warn(
                "Servicio remoto caido, usando cache para usuario {}",
                id);
            Usuario u = cache.get("usr-" + id);
            if (u == null) {
                throw new ServicioNoDisponibleException(
                        "No hay datos de usuario " + id);
            }
            return u;
        }
    }
}
```

Estrategias comunes de degradacion:

- **Cache local**: devolver datos almacenados aunque esten desactualizados.
- **Valores por defecto**: si no se pueden obtener configuraciones, usar valores hardcodeados seguros.
- **Funcionalidad reducida**: deshabilitar funciones no criticas que dependen del servicio caido.
- **Respuesta parcial**: devolver los datos disponibles en lugar de fallar completamente.

## Buenas practicas adicionales

- **No tragar excepciones**: un `catch (IOException e) {}` vacio es una de las peores practicas. Si no se puede manejar, al menos registrar el error.
- **Usar excepciones especificas**: capturar `IOException` en lugar de `Exception` o `Throwable`.
- **Propagar con contexto**: al relanzar, incluir informacion que ayude al diagnostico.

```java
try {
    // operacion
} catch (IOException e) {
    throw new ServicioExternoException(
            "Error al consultar servicio de pagos", e);
}
```

> El objetivo del manejo de errores no es ocultar los fallos, sino responder a ellos de forma predecible y controlada. Una aplicacion de red robusta no es la que nunca falla, sino la que falla sin colapsar.
