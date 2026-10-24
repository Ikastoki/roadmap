# Controller

El Controller es el punto de entrada de la capa de presentacion. Actua como el intermediario entre el mundo exterior (peticiones HTTP, CLI, colas de mensajes) y la logica de la aplicacion. Su unica responsabilidad es recibir la peticion entrante, extraer los parametros necesarios, delegar en la capa de negocio, y retornar una respuesta formateada. Cualquier logica que no sea estrictamente de orquestacion y mapeo pertenece a otra capa.

El Controller no contiene logica de negocio. Esta prohibicion es absoluta: un Controller no decide si un email es valido segun reglas del dominio, no calcula totales, no aplica descuentos. Si lo hiciera, esa logica quedaria atrapada en la capa de presentacion, siendo imposible reutilizarla desde otro punto de entrada (una CLI, un test, un batch) y, peor aun, invisible para cualquier otro componente del sistema que necesite esa misma regla.

## Responsabilidades

Las responsabilidades del Controller se limitan a cinco operaciones concretas:

- **Validar la entrada basica**: verificar que los parametros obligatorios esten presentes, que los tipos sean correctos y que el formato basico sea valido. Esta validacion es superficial; la validacion de reglas de negocio (p. ej., "el email debe ser unico") pertenece al Service.
- **Convertir la request a DTO**: transformar los datos sin estructura de la peticion (parametros HTTP, JSON crudo) en un objeto DTO tipado que la capa de negocio pueda procesar.
- **Llamar al Service correspondiente**: invocar el metodo del Service que implementa la logica necesaria, pasandole el DTO o los parametros ya validados.
- **Convertir la respuesta a DTO**: tomar el objeto devuelto por el Service (normalmente un modelo de dominio o un DTO interno) y convertirlo al formato que el cliente espera.
- **Retornar la respuesta con el codigo de estado adecuado**: empaquetar la respuesta en la estructura estandar de la API y asignar el codigo HTTP correspondiente (200 OK, 201 Created, 404 Not Found, etc.).

## Ejemplo basico

Un Controller minimo pero completo ilustra estas responsabilidades:

```java
public class UserController {
    private final UserService userService;

    public UserController(UserService userService) {
        this.userService = userService;
    }

    public Response getUser(Long id) {
        UserDTO user = userService.findById(id);
        return Response.ok(user).build();
    }

    public Response createUser(CreateUserRequest request) {
        if (request.name() == null || request.name().isBlank()) {
            return Response.badRequest("El nombre es obligatorio").build();
        }
        UserDTO user = userService.createUser(request);
        return Response.created(user).build();
    }

    public Response deleteUser(Long id) {
        userService.deleteById(id);
        return Response.noContent().build();
    }
}
```

Cada metodo del Controller se corresponde con una operacion de la API. El flujo es siempre el mismo: validar entrada, llamar al Service, convertir y retornar. La inyeccion del Service por constructor (en lugar de instanciarlo directamente) permite desacoplar el Controller de la implementacion concreta y facilita el testing con mocks.

## Validacion en el Controller

La validacion en el Controller se limita a comprobaciones de forma: nulabilidad, rangos basicos, formato de datos. No se evaluan reglas de negocio. La separacion es practica: si la regla cambia, solo se modifica el Service; si el formato de entrada cambia, solo se modifica el Controller.

```java
public Response updateUser(Long id, UpdateUserRequest request) {
    if (id == null || id <= 0) {
        return Response.badRequest("ID de usuario invalido").build();
    }
    // La validacion de negocio (ej: email repetido) esta en el Service
    UserDTO user = userService.updateUser(id, request);
    return Response.ok(user).build();
}
```

> Un Controller que filtra, transforma o altera datos segun reglas del dominio esta violando el principio de separacion de responsabilidades. La tentacion de "poner una validacion rapida aqui" es el primer paso hacia un Controller monolitico que termina siendo imposible de testear y mantener.

## Mapeo de rutas y verbos HTTP

En un entorno REST, cada metodo del Controller se asigna a una combinacion de ruta y verbo HTTP. Esta asignacion es responsabilidad del framework (Spring Web, Jakarta REST, Javalin) y no forma parte de la logica del Controller en si. El Controller recibe los parametros ya resueltos por el framework y trabaja con objetos Java, no con la peticion HTTP directamente.

```java
// La anotacion @GetMapping es responsabilidad del framework
@GetMapping("/users/{id}")
public Response getUser(@PathVariable Long id) {
    return userService.findById(id);
}
```

Esta separacion permite que el mismo Controller pueda reutilizarse con diferentes mecanismos de entrada (HTTP, WebSocket, RMI) sin cambiar su implementacion, simplemente cambiando la configuracion del binding.
