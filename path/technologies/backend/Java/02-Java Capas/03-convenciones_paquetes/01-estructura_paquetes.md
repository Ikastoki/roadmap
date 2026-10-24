# Estructura de paquetes

La organizacion del codigo fuente en una aplicacion Java con arquitectura de capas no es un mero detalle estetico: define como los desarrolladores navegan el proyecto, como se gestionan las dependencias entre componentes y, en ultima instancia, que tan sostenible resulta el codigo a largo plazo. Existen dos enfoques principales para estructurar los paquetes, cada uno con implicaciones profundas en el acoplamiento, la cohesion y la escalabilidad del sistema.

## Estructura por capa (technical packaging)

Es el enfoque tradicional y el mas extendido en aplicaciones empresariales Java, especialmente aquellas construidas sobre Spring o Jakarta EE. Los paquetes reflejan la capa tecnica a la que pertenece cada clase, agrupando todos los controladores juntos, todos los servicios juntos y todos los repositorios juntos, independientemente del dominio al que sirvan.

```
com.miapp
├── presentation/
│   ├── controller/
│   │   ├── UserController.java
│   │   └── OrderController.java
│   └── dto/
│       ├── UserDTO.java
│       ├── CreateUserRequest.java
│       └── OrderResponse.java
├── business/
│   ├── service/
│   │   ├── UserService.java
│   │   └── OrderService.java
│   ├── model/
│   │   ├── User.java
│   │   └── Order.java
│   └── exception/
│       ├── UserNotFoundException.java
│       └── OrderValidationException.java
├── data/
│   ├── repository/
│   │   ├── UserRepository.java
│   │   └── OrderRepository.java
│   └── entity/
│       ├── UserEntity.java
│       └── OrderEntity.java
└── config/
    ├── AppConfig.java
    └── SecurityConfig.java
```

**Ventajas**

- Navegacion intuitiva para desarrolladores familiarizados con arquitecturas en capas: sabes que cualquier controller esta en `presentation/controller/`.
- Facilita la aplicacion de cross-cutting concerns como logging, seguridad o transaccionalidad a nivel de capa.
- La estructura es independiente del dominio, lo que la hace predecible en proyectos con muchos modulos.
- Se alinea naturalmente con la inyeccion de dependencias y la configuracion de frameworks (component scanning por paquete).

**Desventajas**

- Cuando el proyecto crece, cada paquete puede acumular decenas o cientos de clases, dificultando localizar todas las clases relacionadas con un mismo dominio.
- Favorece el acoplamiento accidental entre funcionalidades distintas: un cambio en `UserService` puede afectar a `OrderController` porque ambos coexisten en el mismo modulo sin una frontera clara.
- Dificulta la extraccion de modulos o microservicios: separar la funcionalidad de usuarios requiere mover archivos de multiples paquetes.
- La fragmentacion horizontal dificulta tener una vision completa de una funcionalidad concreta.

## Estructura por dominio (domain packaging)

Tambien conocida como packaging por funcionalidad o por modulo. Aqui la primera division del proyecto son los dominios del negocio, y dentro de cada uno se replican las subcapas tecnicas. Cada dominio es autocontenido: contiene su propio controller, su propio service, su propio repository y sus propios DTOs.

```
com.miapp
├── user/
│   ├── presentation/
│   │   ├── UserController.java
│   │   └── UserDTO.java
│   ├── business/
│   │   ├── UserService.java
│   │   ├── User.java
│   │   └── UserNotFoundException.java
│   └── data/
│       ├── UserRepository.java
│       └── UserEntity.java
├── order/
│   ├── presentation/
│   │   ├── OrderController.java
│   │   └── OrderResponse.java
│   ├── business/
│   │   ├── OrderService.java
│   │   ├── Order.java
│   │   └── OrderValidationException.java
│   └── data/
│       ├── OrderRepository.java
│       └── OrderEntity.java
└── shared/
    ├── config/
    │   └── AppConfig.java
    └── util/
        └── DateUtils.java
```

**Ventajas**

- Alta cohesion: todas las clases que participan en una funcionalidad estan en el mismo arbol de directorios.
- Favorece el aislamiento entre modulos: es mas dificil que un cambio en `order/` afecte a `user/` porque cada modulo tiene sus propias abstracciones.
- Facilita la extraccion a microservicios: mover `user/` a un proyecto independiente es cuestion de copiar el directorio.
- Cada equipo puede trabajar en un dominio sin conflictos de naming ni acoplamiento estructural.
- La estructura refleja el lenguaje ubicuo del dominio, alineandose con los principios de Domain-Driven Design.

**Desventajas**

- Puede generar duplicacion de codigo cuando varios dominios comparten logica similar (por ejemplo, todos necesitan un manejo de excepciones basico).
- El componente scanning requiere configuracion mas explicita, ya que las clases de un mismo tipo tecnico estan repartidas en multiples paquetes.
- La navegacion entre capas cruzadas (por ejemplo, encontrar todos los repositorios para una auditoria) requiere recorrer todos los modulos.
- Puede resultar confuso para desarrolladores acostumbrados exclusivamente al enfoque por capa.

## Criterios de eleccion

No existe una opcion universalmente superior; la decision depende del tamano del proyecto, la estructura del equipo y la madurez del dominio.

| Criterio | Por capa | Por dominio |
|---|---|---|
| Proyectos pequenos o prototipos | Recomendado | Sobrecarga innecesaria |
| Proyectos con dominios bien definidos | Funcional | Recomendado |
| Equipos grandes y modulares | Dificulta el aislamiento | Facilita la autonomia |
| Frameworks con component scanning | Alineacion natural | Requiere configuracion adicional |
| Refactorizacion a microservicios | Dificil | Natural |
| Cross-cutting concerns | Facil de aplicar | Requiere shared modules |

> En la practica, muchos proyectos adoptan un enfoque hibrido: utilizan estructura por capa para la infraestructura compartida (config, security, util) y estructura por dominio para los modulos de negocio. Spring Boot, por ejemplo, favorece la estructura por capa por defecto, pero no impide organizar los paquetes por dominio.

## Estructura hibrida

Una alternativa pragmatica consiste en combinar ambos enfoques: los modulos de negocio se organizan por dominio, mientras que los aspectos tecnicos compartidos se mantienen en paquetes transversales.

```
com.miapp
├── shared/
│   ├── config/
│   ├── security/
│   └── util/
├── user/
│   ├── presentation/
│   ├── business/
│   └── data/
└── order/
    ├── presentation/
    ├── business/
    └── data/
```

Esta disposicion ofrece lo mejor de ambos mundos: la cohesion de los dominios y la centralizacion de la infraestructura comun. Es la estructura recomendada para aplicaciones de tamano medio o grande donde los modulos de negocio estan claramente delimitados.
