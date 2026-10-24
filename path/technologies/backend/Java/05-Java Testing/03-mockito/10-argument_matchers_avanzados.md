# Argument matchers avanzados

Cuando los matchers incorporados de Mockito (`any()`, `eq()`, `contains()`, etc.) no expresan adecuadamente la condicion que debe cumplir un argumento, se pueden crear matchers personalizados implementando la interfaz `ArgumentMatcher<T>`. Esta interfaz funcional define un unico metodo `matches(T argument)` que devuelve `true` si el argumento cumple la condicion deseada.

La creacion de un matcher personalizado sigue una estructura simple:

```java
import org.mockito.ArgumentMatcher;

public class UsuarioActivoMatcher implements ArgumentMatcher<Usuario> {

    @Override
    public boolean matches(Usuario usuario) {
        return usuario != null
            && usuario.isActivo()
            && usuario.getEmail() != null
            && usuario.getEmail().contains("@");
    }
}
```

Este matcher se utiliza con `argThat()` en lugar del tipo generico:

```java
when(repositorio.buscar(argThat(new UsuarioActivoMatcher())))
    .thenReturn(listaUsuariosActivos);
```

La interfaz `ArgumentMatcher<T>` es funcional, por lo que se puede reemplazar por una expresion lambda sin necesidad de una clase separada:

```java
when(repositorio.buscar(argThat(usuario ->
    usuario != null
    && usuario.isActivo()
    && usuario.getEmail() != null
    && usuario.getEmail().contains("@")
))).thenReturn(listaUsuariosActivos);
```

Los matchers lambda son convenientes para condiciones simples y locales al test. Cuando la misma logica de matching se repite en multiples pruebas, conviene extraerla a una clase separada para mejorar la reutilizacion y la legibilidad.

La implementacion de `ArgumentMatcher` puede sobrescribir `toString()` para proporcionar mensajes de error descriptivos cuando la verificacion falla:

```java
public class UsuarioActivoMatcher implements ArgumentMatcher<Usuario> {

    @Override
    public boolean matches(Usuario usuario) {
        return usuario != null && usuario.isActivo();
    }

    @Override
    public String toString() {
        return "[Usuario activo con email valido]";
    }
}
```

Mockito usa `toString()` del matcher en los reportes de fallos de verificacion, lo que facilita identificar que condicion no se cumplio.

> Los matchers personalizados deben ser inmutables o al menos stateless. Mockito puede evaluar el mismo matcher multiples veces, y cualquier estado compartido entre evaluaciones producira resultados incorrectos.

Los matchers avanzados tambien permiten composicion. Aunque Mockito no proporciona metodos `and()`/`or()` para matchers de serie, se puede implementar la logica compuesta dentro del metodo `matches()`:

```java
// Composicion manual dentro del matcher
when(repositorio.buscar(argThat(usuario ->
    (usuario.isActivo() || usuario.isVerificado())
    && usuario.getRol() != Rol.INVITADO
))).thenReturn(listaUsuarios);
```

Para condiciones muy complejas, la recomendacion es extraer el matcher a un metodo estatico que devuelva la lambda, manteniendo el codigo de prueba legible:

```java
private static ArgumentMatcher<Usuario> usuarioValido() {
    return usuario -> usuario != null
        && usuario.getEmail() != null
        && usuario.getEmail().matches("^[A-Za-z0-9+_.-]+@(.+)$");
}

// Uso
verify(repositorio).guardar(argThat(usuarioValido()));
```

Esta tecnica se conoce como matchers fluidos y es la misma que Mockito usa internamente para sus matchers predefinidos. El nombre del metodo documenta la intencion de la condicion, mejorando la expresividad del test.

Los matchers personalizados combinados con `argThat()` en verificaciones permiten validar propiedades especificas de los objetos capturados sin recurrir a `ArgumentCaptor`:

```java
verify(repositorio).guardar(argThat(usuario ->
    usuario.getNombre().equals("admin")
    && usuario.getPassword() != null
));
```

La eleccion entre matcher personalizado y `ArgumentCaptor` depende de la necesidad: si solo se requiere validar propiedades, el matcher es mas conciso. Si ademas se necesita el objeto para assert posteriores o para invocar metodos sobre el, `ArgumentCaptor` es la opcion adecuada.
