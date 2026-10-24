# Comparativa de variantes arquitectonicas

Hexagonal, Onion y Clean Architecture comparten una misma tesis: el dominio debe estar aislado de los detalles tecnicos. Las tres invierten las dependencias para que la base de datos, los frameworks y la interfaz de usuario sean reemplazables sin afectar al nucleo del negocio. Sin embargo, difieren en el enfasis, el nivel de prescripcion estructural y la madurez del ecosistema que las rodea. Elegir una sobre otra no es tanto una decision tecnica como una decision de comunicacion y gobierno del codigo.

## Tabla comparativa

| Aspecto | Hexagonal | Onion | Clean Architecture |
|---|---|---|---|
| Origen | Alistair Cockburn (2005) | Jeffrey Palermo (2008) | Robert C. Martin (2012) |
| Mecanismo principal | Puertos y adaptadores | Capas concentricas | Dependency Rule |
| Estructura del interior | Libre (caja negra) | 4 anillos definidos | 4 capas nominales |
| Enfasis | Intercambiabilidad | Organizacion del dominio | Disciplina de dependencias |
| Curva de aprendizaje | Moderada | Moderada | Alta |
| Rigor estructural | Bajo | Medio | Alto |
| Adopcion en industria | Muy alta | Media | Alta |

## Ventajas y desventajas

**Hexagonal** ofrece la maxima flexibilidad: el interior del hexagono puede organizarse como se prefiera, lo que la hace ideal para equipos que ya tienen una estructura interna definida o para proyectos donde la prioridad es la intercambiabilidad de componentes externos. Su desventaja es que proporciona poca guia sobre como estructurar el dominio, lo que puede llevar a inconsistencias cuando varios equipos trabajan sobre la misma base de codigo.

**Onion** proporciona una guia estructural mas clara al dividir el dominio en modelo, servicios de dominio y servicios de aplicacion. Esto ayuda a mantener separadas las reglas de negocio puramente del dominio de la orquestacion de casos de uso. Su principal desventaja es que la proliferacion de anillos puede generar una cantidad excesiva de interfaces y clases para proyectos pequenos.

**Clean Architecture** es la mas prescriptiva de las tres. La nomenclatura estandarizada (entities, use cases, interface adapters, frameworks) facilita que desarrolladores que cambian de proyecto encuentren rapidamente el codigo relevante. La desventaja es que el rigor de los boundaries y la obligacion de mantener modelos separados en cada capa puede resultar en una sobrecarga de mapeo y conversion que en proyectos simples no se justifica.

## Cuando usar cada una

La eleccion depende del tamano del proyecto, la estabilidad del dominio y la madurez del equipo.

**Hexagonal** es la opcion recomendada para la mayoria de proyectos nuevos. Proporciona el desacoplamiento necesario sin imponer una estructura interna rigida. Es especialmente util cuando se anticipa la necesidad de cambiar de base de datos, de framework web o de proveedor de infraestructura, o cuando el equipo ya tiene experiencia en DDD y no necesita una guia adicional sobre como organizar el dominio.

**Onion** encaja bien en proyectos donde el dominio es complejo y se beneficia de la separacion explicita entre modelo de dominio, servicios de dominio y servicios de aplicacion. Es frecuente en proyectos que aplican Domain-Driven Design de forma sistematica y necesitan una estructura que refleje las distinciones semanticas del dominio.

**Clean Architecture** esta indicada para proyectos grandes con multiples equipos, donde la estandarizacion de la estructura del codigo es mas importante que la flexibilidad. Tambien es adecuada cuando se construyen sistemas que deben mantenerse durante decadas, donde el coste de la sobrecarga inicial se amortiza con la claridad estructural a largo plazo. Proyectos gobernados por arquitectos que necesitan hacer cumplir una estructura homogenea en toda la organizacion suelen adoptar Clean Architecture.

> Ninguna de las tres arquitecturas es intrinsecamente superior. Un microservicio con tres casos de uso probablemente se beneficie mas de una hexagonal ligera que de una Clean Architecture con boundaries formales. Un sistema corporativo con cien casos de uso y cinco equipos puede necesitar el rigor de la Clean Architecture para evitar que el codigo degener en un conglomerado incoherente.

## Evolucion historica

La arquitectura hexagonal aparecio primero (2005) como una solucion pragmatica al problema del acoplamiento con la base de datos y los frameworks. Cockburn propuso la metafora del hexagono para evitar la idea de que solo hay dos puertos (entrada y salida) y para enfatizar que la aplicacion puede comunicarse con multiples agentes externos a traves de multiples puertos.

La Onion (2008) llego como una refinamiento que explicitaba la estructura interna del hexagono. Palermo observo que los equipos que adoptaban hexagonal tendian a desorganizar el interior, y propuso los anillos como una guia para mantener la coherencia interna.

La Clean Architecture (2012) sintetizo las lecciones de ambas y las integro con las ideas de Martin sobre diseno de software. Anadio la Dependency Rule como principio autonomo, formalizo los boundaries y estandarizo los nombres de las capas. Desde entonces, los tres enfoques coexisten y se mezclan: es comun ver proyectos que se autodenominan "hexagonales" pero siguen la estructura de capas de Clean Architecture, o proyectos "Clean Architecture" que usan el termino puertos y adaptadores.

En la practica contemporanea, la mayoria de los equipos adoptan una posicion pragmatica: toman el mecanismo de puertos y adaptadores de la hexagonal, la organizacion del dominio de la Onion y la disciplina de dependencias de la Clean Architecture, adaptando el conjunto a las necesidades concretas de su proyecto.
