# Que es Mockito

El testing unitario clasico se enfrenta a un problema fundamental:
las clases rara vez operan en aislamiento.
Una clase tipica depende de repositorios, servicios remotos, colas
de mensajes o sistemas de ficheros.
Probar esa clase sin mockear sus dependencias implica configurar
bases de datos, levantar servidores HTTP o manipular el sistema de
archivos, lo que convierte pruebas unitarias en pruebas de integracion
lentas, fragiles y dificiles de mantener.
Mockito resuelve este problema proporcionando un framework de mocking
para Java que permite crear objetos simulados --mocks-- capaces de
imitar el comportamiento de dependencias reales sin ejecutar su
implementacion.

Mockito opera sobre el principio de sustituir objetos reales por
proxies que registran cada interaccion y responden segun configuracion
explicita.
Esto permite aislar la unidad bajo prueba (SUT, System Under Test) y
verificar que colabora correctamente con sus dependencias sin importar
como estas esten implementadas.
El resultado son pruebas deterministas, rapidas y repetibles que fallan
solo cuando la logica de la SUT es incorrecta, no cuando falla la
infraestructura.

La necesidad de mocking surge de la arquitectura misma del software
orientado a objetos.
Una clase que instancia directamente sus dependencias con `new` es
dificil de probar; una que las recibe por constructor (inyeccion de
dependencias) puede sustituirlas por mocks en el contexto de prueba.
Mockito explota este patron y ofrece una API fluida, legible y
minimalista que se integra sin friccion con JUnit 5, TestNG y otros
runners.

> Mockito no es un framework de pruebas en si mismo; es una herramienta
> de aislamiento que se combina con un runner de pruebas como JUnit 5
> para construir el escenario de test.

Mockito compite historica y tecnicamente con EasyMock y PowerMock.
EasyMock fue el pionero en el espacio de mocking para Java, pero su API
basada en record/replay resulta menos intuitiva que el estilo
declarativo de Mockito.
PowerMock extiende tanto Mockito como EasyMock permitiendo mockear
constructores, metodos estaticos y final, algo que Mockito nativamente
no soporta.
Sin embargo, PowerMock requiere modificaciones en el classloader y su
uso suele indicar un diseno que necesita refactorizacion.
Mockito, por su parte, ha evolucionado incorporando soporte parcial
para metodos estaticos y final desde su version 3.x, reduciendo
progresivamente la necesidad de PowerMock.

Las ventajas principales de Mockito frente a alternativas incluyen:
sintaxis legible con metodos como `when().thenReturn()` que reflejan
Arrange-Act-Assert, verificacion explicita de interacciones con
`verify()`, integracion nativa con JUnit 5 via
`@ExtendWith(MockitoExtension.class)`, creacion automatica de mocks
con anotaciones, y un modelo de espias que permite combinar
comportamiento real con stubbing parcial.
La comunidad amplia y la madurez del proyecto (20+ anos de desarrollo
activo) consolidan Mockito como el estandar de facto para mocking en
Java.
