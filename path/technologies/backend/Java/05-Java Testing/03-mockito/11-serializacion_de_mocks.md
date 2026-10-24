# Serializacion de mocks

En ciertos escenarios, un mock debe atravesar los limites de serializacion de Java. Esto ocurre tipicamente cuando el objeto bajo prueba se despliega en un contenedor que serializa sus componentes (como sesiones HTTP en aplicaciones web, RMI, o frameworks de cache distribuida) o cuando el mock se utiliza en pruebas que involucran serializacion como parte del flujo.

Por defecto, los mocks creados con `Mockito.mock()` no son serializables. Intentar serializarlos lanza `NotSerializableException`. Para crear un mock serializable se utiliza `MockSettings` con la opcion `serializable()`:

```java
Usuario mockSerializable = Mockito.mock(Usuario.class,
    Mockito.withSettings().serializable());
```

El mock resultante implementa `Serializable` y puede serializarse y deserializarse como cualquier objeto Java. Los stubs configurados en el mock original se preservan a traves de la serializacion porque Mockito almacena las configuraciones de stubbing en objetos que tambien son serializables.

Para mayor control, se puede especificar un `serializableMode()`:

```java
Usuario mock = Mockito.mock(Usuario.class,
    Mockito.withSettings().serializable(SerializableMode.ACROSS_CLASSLOADERS));
```

`SerializableMode.ACROSS_CLASSLOADERS` permite que el mock se serialize y deserialize incluso cuando los classloaders del emisor y receptor son distintos, algo comun en entornos de aplicaciones empresariales y contenedores.

La serializacion preserva:

- Los stubs configurados con `when().thenReturn()` y variantes.
- El estado interno del mock, incluyendo registros de invocaciones.

No se preservan a traves de la serializacion:

- Las referencias a objetos no serializables dentro de los stubs. Si un `thenReturn()` devuelve un objeto que no es `Serializable`, la serializacion fallara en tiempo de ejecucion.
- Los callbacks y `Answer` personalizados que no implementan `Serializable`.

```java
// Esto falla al serializar si ResultadoNoSerializable no es Serializable
when(servicio.ejecutar()).thenReturn(new ResultadoNoSerializable());

// Esto funciona si el resultado es Serializable
when(servicio.ejecutar()).thenReturn(new ResultadoSerializable());
```

> Para que un mock serializable sea realmente util, todos los objetos involucrados en los stubs deben ser serializables. Esto incluye los argumentos de `when()`, los valores de retorno, y las excepciones en `thenThrow()`.

La necesidad de mocks serializables surge en contextos especificos:

- Pruebas de componentes que se almacenan en `HttpSession` y deben serializarse para replicacion en cluster.
- Frameworks que cachean resultados en memoria distribuida (Hazelcast, Redis) y serializan objetos.
- Pruebas que verifican el comportamiento de RMI o servicios remotos que intercambian objetos serializados.
- Testing de aplicaciones Spring que usan ambitos de sesion o peticion con serializacion implicita.

Si el caso de uso no involucra serializacion real, no es necesario crear mocks serializables. La opcion `serializable()` anade complejidad y restricciones sin beneficio. La mayoria de las pruebas unitarias no requieren esta caracteristica; queda reservada para pruebas de integracion o sistemas distribuidos donde la serializacion forma parte del contrato entre componentes.
