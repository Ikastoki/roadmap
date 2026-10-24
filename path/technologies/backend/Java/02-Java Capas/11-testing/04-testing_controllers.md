# Testing de controllers

Los controllers son la puerta de entrada de la aplicacion. Probar esta capa significa verificar que las peticiones HTTP se procesan correctamente: que los parametros se extraen, que los servicios se invocan con los datos adecuados, que las respuestas tienen el formato y el codigo de estado esperados, y que los errores se manejan de forma consistente.

La capa de presentacion no contiene logica de negocio, por lo que los tests de controllers se centran en la orquestacion y el mapeo. El servicio subyacente se mockea para aislar el controller y hacer que los tests sean deterministicos y rapidos.

## Mock de servicios

Mockear el servicio permite controlar que devuelve (un DTO, una excepcion, null) sin necesidad de ejecutar la logica real. Spring Boot proporciona `@WebMvcTest` para cargar solo los beans de la capa web y `@MockBean` para sustituir dependencias concretas por mocks.

```java
@WebMvcTest(UserController.class)
class UserControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private UserService userService;

    @Test
    void shouldReturnUserWhenFound() throws Exception {
        UserDTO userDTO = new UserDTO(1L, "Ana", "ana@example.com");
        when(userService.findById(1L)).thenReturn(userDTO);

        mockMvc.perform(get("/api/users/1"))
               .andExpect(status().isOk())
               .andExpect(jsonPath("$.id").value(1))
               .andExpect(jsonPath("$.name").value("Ana"))
               .andExpect(jsonPath("$.email").value("ana@example.com"));

        verify(userService).findById(1L);
    }

    @Test
    void shouldReturn404WhenUserNotFound() throws Exception {
        when(userService.findById(99L)).thenThrow(new NotFoundException("User", 99L));

        mockMvc.perform(get("/api/users/99"))
               .andExpect(status().isNotFound())
               .andExpect(jsonPath("$.message").value("User not found with id: 99"));
    }
}
```

`MockMvc` simula el flujo completo de una peticion HTTP sin levantar un servidor real. Envia la peticion, la procesa a traves de la cadena de filtros y handlers de Spring MVC, y permite inspeccionar la respuesta. Es mas rapido que un `TestRestTemplate` porque no require puerto TCP, y suficiente para la mayoria de tests de controller.

> `@WebMvcTest` carga exclusivamente los beans de la capa web: controllers, exception handlers, filtros y convertidores. No carga servicios, repositorios ni la configuracion de base de datos, lo que reduce drasticamente el tiempo de arranque.

## Simular peticiones HTTP

`MockMvc` acepta builders para configurar cualquier aspecto de la peticion HTTP: metodo, URI, headers, parametros de query, cuerpo JSON y contenido de formulario.

```java
@Test
void shouldCreateUser() throws Exception {
    CreateUserRequest request = new CreateUserRequest("Ana", "ana@example.com");
    UserDTO userDTO = new UserDTO(1L, "Ana", "ana@example.com");
    when(userService.createUser(any())).thenReturn(userDTO);

    mockMvc.perform(post("/api/users")
               .contentType(MediaType.APPLICATION_JSON)
               .content("{\"name\":\"Ana\",\"email\":\"ana@example.com\"}"))
           .andExpect(status().isCreated())
           .andExpect(header().exists("Location"))
           .andExpect(jsonPath("$.id").value(1));

    ArgumentCaptor<CreateUserRequest> captor = ArgumentCaptor.forClass(CreateUserRequest.class);
    verify(userService).createUser(captor.capture());
    assertEquals("Ana", captor.getValue().name());
    assertEquals("ana@example.com", captor.getValue().email());
}
```

El uso de `ArgumentCaptor` permite inspeccionar el objeto exacto que el controller paso al servicio, confirmando que el mapeo del JSON de entrada al DTO se realizo correctamente.

Para peticiones con parametros de query:

```java
@Test
void shouldListUsersWithPagination() throws Exception {
    Page<UserDTO> page = new PageImpl<>(List.of(
        new UserDTO(1L, "Ana", "ana@example.com")
    ));
    when(userService.findAll(any(Pageable.class))).thenReturn(page);

    mockMvc.perform(get("/api/users")
               .param("page", "0")
               .param("size", "10")
               .param("sort", "name,asc"))
           .andExpect(status().isOk())
           .andExpect(jsonPath("$.content.length()").value(1))
           .andExpect(jsonPath("$.content[0].name").value("Ana"));

    ArgumentCaptor<Pageable> captor = ArgumentCaptor.forClass(Pageable.class);
    verify(userService).findAll(captor.capture());
    assertEquals(0, captor.getValue().getPageNumber());
    assertEquals(10, captor.getValue().getPageSize());
}
```

## Verificar respuestas HTTP

Las respuestas HTTP deben comprobarse en tres dimensiones: codigo de estado, headers y cuerpo. Para cada escenario (exito, error de validacion, error de negocio, error interno) el controller debe devolver la respuesta estandarizada que la API promete.

```java
@Test
void shouldReturn400WhenValidationFails() throws Exception {
    mockMvc.perform(post("/api/users")
               .contentType(MediaType.APPLICATION_JSON)
               .content("{\"name\":\"\",\"email\":\"invalido\"}"))
           .andExpect(status().isBadRequest())
           .andExpect(jsonPath("$.errors").isArray())
           .andExpect(jsonPath("$.errors[0].field").exists());
}

@Test
void shouldReturn400WhenBusinessRuleFails() throws Exception {
    when(userService.createUser(any()))
        .thenThrow(new BusinessException("El email ya esta en uso"));

    mockMvc.perform(post("/api/users")
               .contentType(MediaType.APPLICATION_JSON)
               .content("{\"name\":\"Ana\",\"email\":\"existente@example.com\"}"))
           .andExpect(status().isBadRequest())
           .andExpect(jsonPath("$.message").value("El email ya esta en uso"));
}

@Test
void shouldReturn500OnUnexpectedError() throws Exception {
    when(userService.findById(any()))
        .thenThrow(new RuntimeException("Error inesperado"));

    mockMvc.perform(get("/api/users/1"))
           .andExpect(status().isInternalServerError());
}
```

La estructura de la respuesta debe ser consistente en todos los casos. Un formato tipico incluye un campo `success` booleano, un `message` descriptivo y un campo `data` opcional:

```json
{
    "success": true,
    "data": { "id": 1, "name": "Ana", "email": "ana@example.com" },
    "timestamp": "2026-07-28T12:00:00"
}
```

```json
{
    "success": false,
    "message": "El email ya esta en uso",
    "timestamp": "2026-07-28T12:00:00"
}
```

El `@ControllerAdvice` o el manejador global de excepciones se encarga de esta transformacion. Los tests de controller verifican que el manejador responde correctamente para cada tipo de excepcion, manteniendo la API coherente incluso en situaciones de error.

Los tests de la capa de presentacion cierran el circulo: confirman que la entrada externa se transforma en llamadas al servicio y que las respuestas del servicio se transforman en respuestas HTTP correctas. Como los servicios ya tienen sus propios tests (unitarios con mocks y de integracion con repositorios reales), el test de controller puede centrarse en su unica responsabilidad: la orquestacion HTTP.
