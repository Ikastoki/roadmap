# Auditoria de Seguridad

La auditoria de seguridad es el proceso sistematico de evaluar la postura de seguridad de una organizacion, identificando debilidades, vulnerabilidades y desviaciones respecto a las politicas establecidas. No se trata de un evento puntual sino de un ciclo continuo que permite mantener una visibilidad constante sobre el estado real de las defensas. Sin auditorias regulares, una organizacion opera a ciegas: no sabe si sus controles funcionan, si ha surgido una nueva superficie de ataque o si los cambios recientes en la infraestructura han introducido riesgos no detectados. La auditoria de seguridad proporciona la evidencia objetiva necesaria para tomar decisiones informadas sobre inversiones en seguridad, priorizar remediaciones y demostrar cumplimiento normativo.

## Evaluacion de vulnerabilidades

Una evaluacion de vulnerabilidades consiste en el escaneo periodico de la infraestructura con el objetivo de catalogar todas las debilidades conocidas en sistemas operativos, aplicaciones, configuraciones de red y servicios expuestos. El proceso utiliza bases de datos de vulnerabilidades como CVE (Common Vulnerabilities and Exposures) para correlacionar los hallazgos con fallos documentados y sus respectivos identificadores.

La evaluacion periodica es fundamental porque el panorama de amenazas cambia constantemente. Nuevas vulnerabilidades se publican a diario, parches de seguridad se liberan semanalmente y la infraestructura de una organizacion evoluciona con cada despliegue o cambio de configuracion. Un escaneo realizado hace tres meses no garantiza que el sistema sea seguro hoy.

El flujo tipico de una evaluacion de vulnerabilidades incluye:

1. **Descubrimiento**: se identifican todos los activos conectados a la red, incluyendo dispositivos que pudieran ser desconocidos para el equipo de TI.
2. **Escaneo**: se ejecutan plugins y firmas contra cada activo para detectar versiones de software, configuraciones inseguras, puertos abiertos y servicios con debilidades conocidas.
3. **Analisis**: se cruzan los resultados del escaneo con la criticidad de los activos y el contexto del negocio para determinar el riesgo real.
4. **Remediacion**: se aplican parches, se reconfiguran servicios o se implementan compensaciones temporales.
5. **Verificacion**: se re-escanea para confirmar que las correcciones han sido efectivas.

> Es importante distinguir entre severidad tecnica y riesgo de negocio. Una vulnerabilidad criticamente clasificada en un servidor de pruebas aislado puede representar menos riesgo que una vulnerabilidad moderada en el servidor de base de datos de produccion que almacena datos de clientes.

## Pruebas de penetracion

Una prueba de penetracion (pentest) va mas alla del escaneo automatico: un profesional especializado intenta explotar activamente las vulnerabilidades encontradas para determinar si un atacante real podria comprometer el sistema. Mientras que la evaluacion de vulnerabilidades es un proceso en su mayor parte automatizado que responde a la pregunta "que debilidades existen", la prueba de penetracion responde a la pregunta "pueden estas debilidades ser efectivamente explotadas para obtener acceso o exfiltrar datos".

### Diferencia entre escaneo de vulnerabilidades y prueba de penetracion

| Aspecto | Escaneo de vulnerabilidades | Prueba de penetracion |
|---|---|---|
| **Metodo** | Automatico, basado en firmas y plugins | Manual con herramientas de apoyo |
| **Profundidad** | Superficial: identifica debilidades conocidas | Profunda: intenta explotar y encadenar debilidades |
| **Alcance** | Amplio: cubre toda la infraestructura | Focalizado: objetivos especificos definidos |
| **Frecuencia** | Periodico (mensual, trimestral) | Periodico o ante cambios significativos |
| **Resultado** | Lista de vulnerabilidades clasificadas por severidad | Narrativa del ataque con pasos ejecutados y evidencia |
| **Coste** | Relativamente bajo | Significativamente mayor |
| **Simulacion** | No模拟a un ataque real | Simula el comportamiento de un atacante |

Un escaneo de vulnerabilidades puede detectar que un servidor ejecuta una version vulnerable de OpenSSL, pero no determinara si esa vulnerabilidad es realmente explotable en el contexto de la arquitectura de red de la organizacion, ni intentara encadenarla con otros descubrimientos para obtener privilegios elevados. La prueba de penetracion si lo hace.

> Las organizaciones suelen combinar ambos enfoques: escaneos automatizados frecuentes para mantener la cobertura y pentest trimestrales o anuales para validar la efectividad real de las defensas.

## Benchmark y CIS Benchmarks

Un benchmark de seguridad es un estandar de referencia que define las configuraciones correctas y seguras para sistemas operativos, servicios de red, bases de datos y aplicaciones. Los CIS Benchmarks (Center for Internet Security) son los estandares de referencia mas ampliamente utilizados en la industria. Estos documentos proporcionan configuraciones especificas, paso a paso, para endurecer sistemas segun las mejores practicas de seguridad.

La importancia de los CIS Benchmarks radica en que transforman recomendaciones abstractas de seguridad en configuraciones concretas y verificables. En lugar de decir "el servidor debe ser seguro", los CIS Benchmarks especifican exactamente que parametro debe configurarse, que valor debe tener y por que ese valor es el adecuado. Esto elimina la ambiguidad y permite la verificacion automatizada del cumplimiento.

Algunos ejemplos de areas que cubren los CIS Benchmarks:

- **Sistemas operativos**: deshabilitar servicios innecesarios, configurar politicas de contrasenas, establecer permisos de archivos criticos, habilitar el registro de auditoria.
- **Servidores web**: configurar headers de seguridad HTTP, deshabilitar metodos HTTP peligrosos, establecer cifrado TLS adecuado.
- **Bases de datos**: restringir acceso a usuarios privilegiados, habilitar el cifrado en reposo, configurar politicas de respaldo seguras.
- **Equipos de red**: deshabilitar puertos no utilizados, configurar listas de control de acceso, habilitar autenticacion en protocolos de gestion.

## Herramientas de auditoria

### OpenVAS

OpenVAS (Open Vulnerability Assessment Scanner) es un framework abierto de escaneo de vulnerabilidades que incluye un motor de escaneo, un amplio conjunto de plugins y una interfaz de gestion. Es la herramienta de referencia en el ecosistema de codigo abierto para evaluacion de vulnerabilidades. Su base de datos de vulnerabilidades (Greenbone Vulnerability Feed) se actualiza periodicamente con nuevas firmas de deteccion. OpenVAS permite programar escaneos, generar informes automatizados y clasificar hallazgos por severidad y metricas CVSS.

### Nessus

Nessus, desarrollado por Tenable, es probablemente la herramienta comercial de escaneo de vulnerabilidades mas extendida en la industria. Ofrece escaneos agentes y agentless, politicas de escaneo personalizables, integracion con sistemas de gestion de activos y capacidades de compliance scanning que permiten verificar el cumplimiento de estandares como los CIS Benchmarks. Nessus Professional es la edicion para organizaciones, mientras que Nessus Essentials ofrece una version gratuita limitada a 16 direcciones IP.

### Lynis

Lynis es una herramienta de auditoria de seguridad para sistemas basados en Unix (Linux, macOS, BSD) que funciona de forma local, sin necesidad de agentes remotos. A diferencia de OpenVAS y Nessus, que se centran en vulnerabilidades de red y servicios expuestos, Lynis audita la configuracion interna del sistema operativo: permisos de archivos criticos, configuracion del kernel, permisos de usuarios, registros de auditoria y politicas de contrasenas. Es ideal para el endurecimiento (hardening) de servidores y para verificar el cumplimiento de CIS Benchmarks a nivel local.

```bash
lynis audit system
```

> Lynis no detecta vulnerabilidades de red ni explota servicios remotos. Es complementario a herramientas como Nessus u OpenVAS, que operan desde la perspectiva del atacante externo.

---

