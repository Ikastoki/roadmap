# Principios generales de la arquitectura de capas

Detras de la estructura visible --paquetes, nombres de clase, anotaciones-- subyacen principios mas profundos que definen si una implementacion es correcta o solo aparenta serlo. La arquitectura de capas no es una receta de carpetas, sino un conjunto de restricciones que gobiernan las dependencias y el flujo de control. Comprender estos principios es lo que separa a un desarrollador que sigue una moda de otro que disena con intencion.

## Unidireccionalidad

El principio mas rigido y, paradójicamente, el que mas se viola en la practica. Las dependencias deben fluir en una sola direccion: de arriba hacia abajo. Una capa superior puede depender de una capa inferior, pero la inferior jamas debe depender de la superior. Esto significa que la capa de persistencia no puede importar clases de la capa de negocio, y la capa de negocio no puede importar clases de la capa de presentacion.

La violacion mas comun ocurre cuando la capa de persistencia devuelve objetos que contienen logica de negocio o, peor aun, cuando la capa de negocio pasa referencias de la capa de presentacion (como `HttpServletRequest`) a capas inferiores. Una vez que un objeto `HttpServletRequest` aparece en un servicio, todo el arbol de dependencias queda contaminado con la API de servlets. La regla es clara: las capas inferiores solo conocen tipos definidos en su propio nivel o en niveles aun mas bajos.

```java
// Bien: el servicio solo conoce su propia capa y la de persistencia
public class PedidoService {
    private PedidoRepository repository;
    public Pedido crearPedido(DatosPedido datos) { ... }
}

// Mal: el servicio conoce detalles de presentacion
public class PedidoService {
    public Pedido crearPedido(HttpServletRequest request) { ... }
}
```

## Independencia

Cada capa debe poder desarrollarse, probarse y evolucionar con autonomia respecto a las demas. La independencia se consigue a traves de contratos estables: interfaces bien definidas que actuan como fronteras. La capa de presentacion solo conoce las interfaces de los servicios; ignora que implementacion concreta se ejecuta detras. La capa de negocio solo conoce las interfaces de los repositorios; no sabe si los datos residen en MySQL, Redis o un archivo plano.

Esta independencia tiene consecuencias practicas inmediatas. Permite cambiar la implementacion de una capa sin modificar las otras, siempre que los contratos se respeten. Permite probar cada capa de forma aislada. Permite, incluso, que diferentes equipos trabajen en paralelo sobre capas distintas mientras acuerdan las interfaces por anticipado.

En Java, la independencia se materializa con frecuencia a nivel de modulo. Maven o Gradle permiten definir modulos separados para cada capa, cada uno con sus propias dependencias y ciclo de release. El modulo de dominio, por ejemplo, puede definir las entidades y las interfaces de repositorio sin incluir ninguna dependencia de framework o base de datos.

> La independencia no es gratuita: exige disciplina en la definicion de interfaces y un costo inicial de diseno. Pero cada hora invertida en definir bien una interfaz ahorra dias de refactorizacion cuando los requisitos cambian.

## Abstraccion

La capa inferior nunca se expone directamente a la capa superior. Entre ambas media una abstraccion que oculta los detalles de implementacion. Esta abstraccion es tipicamente una interfaz Java que define las operaciones disponibles sin revelar como se ejecutan. La capa superior programa contra la interfaz, no contra la implementacion concreta.

El principio de abstraccion se extiende a los tipos de datos que cruzan las fronteras entre capas. Es buena practica definir objetos de transferencia de datos (DTO) especificos para cada capa, impidiendo que las entidades de persistencia (con sus anotaciones JPA, lazy loading y relaciones) se filtren hacia la capa de presentacion. Un `@Entity` de JPA que llega a la capa de presentacion acopla la vista al modelo de base de datos y, peor aun, expone problemas de lazy loading fuera del contexto transaccional.

```java
// Abstraccion: la capa superior usa un DTO desacoplado
public class UsuarioDTO {
    private String nombre;
    private String email;
}

// La implementacion concreta permanece oculta
public class UsuarioJpaRepository implements UsuarioRepository {
    @Override
    public UsuarioDTO buscarPorId(Long id) {
        // Traduce UsuarioEntity a UsuarioDTO
    }
}
```

La abstraccion bien aplicada produce un efecto de "navegacion por capas": al leer el codigo de la capa de presentacion se entiende la interaccion sin conocer los detalles de persistencia; al leer la capa de persistencia se entiende el almacenamiento sin conocer la interfaz de usuario. Cada nivel cuenta su propia historia, y la suma de todas ellas es el sistema completo.
