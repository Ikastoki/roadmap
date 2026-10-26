# Seguridad de la Cadena de Suministro

La cadena de suministro de software es el conjunto de todas las dependencias, herramientas, plataformas y procesos que intervienen desde que un desarrollador escribe codigo hasta que el producto final llega al usuario. Cada eslabon de esa cadena — desde la libreria de utilidades descargada de un repositorio de paquetes hasta el compilador que genera el binario — representa un punto potencial de compromiso. Un atacante no necesita vulnerar la aplicacion final; le basta con inyectar codigo malicioso en cualquiera de las piezas que componen el ecosistema de construccion. La seguridad de la cadena de suministro no es un dominio aislado del DevOps: es la extension logica de la defensa en profundidad hacia el ecosistema externo sobre el que el software depende.

## El ataque SolarWinds: un estudio de caso

En diciembre de 2020, se descubrió que SolarWinds, proveedor de software de gestion de infraestructura utilizado por miles de organizaciones gubernamentales y corporativas, habia sido victima de un ataque de cadena de suministro de proporciones sin precedentes. Los atacantes — atribuidos al grupo APT29 (Cozy Bear) del SVR ruso — lograron inyectar codigo malicioso en el proceso de compilacion del producto Orion, modificando la herramienta de build para que insertara una puerta trasera (denominada SUNBURST) en las actualizaciones oficiales del software.

El mecanismo fue particularmente sofisticado: en lugar de comprometer el repositorio de codigo fuente directamente (lo que habria sido detectable en revisiones de codigo), los atacantes modificaron el compilador o los scripts de build, de modo que el codigo fuente original era limpio pero el artefacto final contenia codigo malicioso. Las actualizaciones comprometidas se distribuyeron a traves del mecanismo oficial de actualizacion de SolarWinds, lo que significa que las victimas instalaron voluntariamente la version backdoored confiando en la integridad de la cadena de distribucion.

El impacto fue devastador: aproximadamente 18.000 organizaciones instalaron la actualizacion comprometida, incluyendo agencias del gobierno de Estados Unidos como el Tesoro, el Departamento de Comercio y el Departamento de Seguridad Nacional. El software malicioso permanecio indetectado durante nueve meses. Este ataque demostro que la confianza ciega en la cadena de suministro de software es una vulnerabilidad critica y que los controles de seguridad deben extenderse mas alla del codigo fuente propio hacia todas las piezas externas que componen el ecosistema de construccion y distribucion.

## Dependencias de terceros y riesgo de ataque

### La paradoja de la productividad

El desarrollo moderno se construye sobre una base masiva de dependencias de terceros. Un proyecto tipico de Node.js puede incorporar cientos o miles de paquetes a traves de npm; una aplicacion Python depende de PyPI; un microservicio en Go incorpora modulos de github.com. Esta reutilizacion es lo que hace posible la velocidad de desarrollo actual, pero crea simultaneamente una superficie de ataque difusa y dificil de auditar. Cada paquete importado trae consigo su propio historial de seguridad, sus propios mantenedores, y sus propias dependencias transitivas.

### Tipos de ataques a la cadena de suministro

Los ataques a la cadena de suministro de software se clasifican en varias categorias segun el punto de compromiso:

**Compromiso de repositorio de paquetes.** El atacante publica un paquete malicioso en un registry publico (npm, PyPI, RubyGems) aprovechando un nombre similar a una libreria popular (typosquatting) o secuestrando una cuenta de mantenedor de un paquete existente. Ejemplos reales incluyen `event-stream` (un paquete npm popular que fue comprometido en 2018 cuando un mantenedor voluntario cedo la mantenencia a un atacante) y los multiples paquetes de PyPI que imitan bibliotecas populares con codigo que roba credenciales.

**Compromiso de dependencia transitiva.** Un paquete directo puede ser legitimo, pero alguna de sus dependencias — que el desarrollador no importo explicitamente — ha sido comprometida. Dado que las dependencias transitivas se resuelven automaticamente por el gestor de paquetes, el codigo malicioso se ejecuta sin que el desarrollador lo sepa ni lo autorice.

**Compromiso del entorno de construccion.** Como demostro SolarWinds, el atacante puede comprometer los propios herramientos de compilacion: el compilador, los scripts de build, las herramientas de empaquetado. En este escenario, incluso si el codigo fuente es verificable y limpio, el artefacto resultante contiene codigo que nunca existio en el repositorio.

**Compromiso de la infraestructura CI/CD.** El atacante accede a los runners, los servidores de Jenkins o los agentes de compilacion para manipular el proceso de build directamente, inyectando pasos maliciosos o modificando el artefacto generado antes de publicarlo.

**Compromiso del canal de distribucion.** El artefacto legitimno es interceptado y reemplazado en el mirror, registry o repositorio de descarga antes de llegar al usuario final. Ataques de tipo man-in-the-middle contra servidores de actualizacion o repositorios que no utilizan verificacion criptografica caen en esta categoria.

## SBOM (Software Bill of Materials)

### Definicion y proposito

Un SBOM es un inventario estructurado y machine-readable de todos los componentes que componen un software: librerias directas, dependencias transitivas, versiones exactas, proveedores, identificadores de licenciamiento y relaciones de dependencia entre componentes. Piensa en el SBOM como la lista de ingredientes de un alimento industrializado: sin ella, es imposible determinar si el producto contiene alergenos, ingredientes prohibidos o componentes de origen cuestionable.

El SBOM se convierte en una herramienta de seguridad critica cuando se combina con bases de datos de vulnerabilidades. Si se publica un CVE para la version 1.3.2 de una libreria, una organizacion con un SBOM actualizado puede identificar inmediatamente todos los productos, servicios y componentes que utilizan esa version afectada y priorizar la remediacion. Sin un SBOM, la respuesta a un CVE en una dependencia de terceros es un proceso manual, lento e incompleto.

### Formatos estandar

Existen dos formatos principales de SBOM que han ganado traccion en la industria:

**SPDX (Software Package Data Exchange).** Desarrollado originalmente por la Linux Foundation y normalizado como ISO/IEC 5962:2021, SPDX es el formato mas maduro y completo. Define una estructura que incluye no solo los componentes y sus versiones, sino tambien las relaciones entre ellos, las licencias aplicables, los hashes criptograficos de cada componente y la informacion de provenance.

**CycloneDX.** Creado por OWASP, CycloneDX esta diseñado especificamente para aplicarse a los flujos de trabajo de seguridad de la cadena de suministro. Ademas de los campos basicos de componente, soporta la inclusion de vulnerabilidades conocidas, la informacion de dependencias y metadatos de privacidad. CycloneDX tiende a ser preferido en entornos donde la seguridad es la prioridad principal, mientras que SPDX se usa mas en contextos de cumplimiento de licencias.

### Generacion y uso del SBOM

El SBOM se genera automaticamente durante el proceso de compilacion, extrayendo la informacion de los gestores de dependencias del proyecto. Herramientas como Syft (para imagenes Docker y sistemas de archivos), CycloneDX-CLI o `npm bill-of-materials` pueden generar SBOMs en los formatos estandar. En un pipeline tipico, el SBOM se genera despues de la resolucion de dependencias y se adjunta al artefacto como metadata, permitiendo que cualquier consumidor verifique la composicion exacta del software.

> El Executive Order 14028 del gobierno de Estados Unidos (2022) exige que los proveedores de software vendido al gobierno proporcionen un SBOM junto con sus productos. Esto ha convertido la generacion de SBOMs de una recomendacion de mejores practicas a un requisito regulatorio en muchos contextos.

## SLSA (Supply-chain Levels for Software Assurance)

### Origen y motivacion

SLSA es un framework de seguridad de la cadena de suministro creado por Google en respuesta directa al ataque SolarWinds. Su objetivo es establecer un conjunto de estandares y controles verificables que protejan contra la manipulacion de software en cada etapa del ciclo de vida, desde la fuente hasta el despliegue. SLSA no es un producto ni una herramienta; es un marco de referencia que define niveles de madurez progresivos, donde cada nivel incorpora los controles del anterior y añade nuevas protecciones.

### Niveles de SLSA

SLSA define cuatro niveles de seguridad, ordenados de menor a mayor proteccion:

**Nivel 1: Construccion documentada.** El software se construye mediante algun proceso automatizado y se genera un SBOM basico. Este nivel establece la trazabilidad minima: se sabe que el software fue construido por algun proceso, pero no se garantiza que ese proceso no haya sido manipulado. Equivale a tener la factura de un producto sin poder verificar su cadena de custody.

**Nivel 2: Proceso protegido.** La compilacion se ejecuta en una plataforma de CI/CD gestionada (no en la maquina del desarrollador) que genera provenance firmada: un registro criptografico que certifica que el artefacto fue generado por un pipeline especifico, en una plataforma especifica, a partir de un commit especifico. Herramientas como GitHub Actions con SLSA Generator, Google Cloud Build o GitLab CI con attestations pueden producir provenance de nivel 2. Este nivel protege contra la contaminacion individual de un desarrollador pero no contra la compromiso de la plataforma de build en si.

**Nivel 3: Build hardening.** La plataforma de compilacion incorpora controles de seguridad avanzados: aislamiento entre ejecuciones, reproducibilidad garantizada (el mismo fuente produce el mismo binario), y proteccion contra la manipulacion incluso por parte de administradores de la plataforma. Este nivel busca mitigar ataques tipo SolarWinds, donde el compromiso del propio entorno de compilacion es el vector de ataque.

**Nivel 4: Servicio resistente a la hostilidad.** El nivel mas alto de SLSA requiere que la fuente y el historial de cambios sean maintainidos por dos o mas personas independientes (two-party review), que los commits esten protegidos contra reescritura del historial, y que la infraestructura de compilacion este en un entorno altamente controlado y monitoreado. Este nivel esta disenado para software critico de infraestructura donde un compromiso tendria consecuencias sistemicas.

### Ataque a la fuente y SLSA

Uno de los vectores de ataque que SLSA busca mitigar es el de la "fuente compartida": cuando el mismo repositorio de codigo fuente es utilizado por multiples productos o equipos, un compromiso del repositorio afecta a toda la cadena. SLSA mitiga este riesgo requiriendo que la provenance vincule cada artefacto a un commit especifico y que los mecanismos de branching y merging esten protegidos contra reescritura, haciendo que un atacante no pueda modificar retroactivamente el historial de cambios sin ser detectado.

## Fijacion de versiones y hashing de dependencias

### El riesgo de las versiones flexibles

Los gestores de dependencias permiten especificar rangos de versiones compatibles en los manifiestos del proyecto. En npm, por ejemplo, `^1.3.0` significa "cualquier version compatible con 1.3.0 (>=1.3.0 <2.0.0)"; en Cargo (Rust), `1.3` acepta 1.3.0, 1.3.1, 1.3.2, y asi sucesivamente. Esta flexibilidad es util para recibir correcciones de seguridad y mejoras sin cambiar el manifiesto, pero introduce un riesgo critico: en el momento de la instalacion, el gestor de paquetes resuelve las versiones disponibles y descarga la ultima compatible. Si un atacante publica una version maliciosa de una dependencia dentro del rango permitido, el pipeline la instalara automaticamente sin notificacion.

### Bloqueo de versiones (lock files)

La contramedida principal es el uso de archivos de bloqueo de versiones: `package-lock.json` en npm, `yarn.lock` en Yarn, `poetry.lock` en Python Poetry, `Cargo.lock` en Rust, `go.sum` en Go. Estos archivos registran la version exacta de cada dependencia (directa y transitiva) que fue resuelta en el momento de generar el lock file, junto con un hash criptografico de su contenido. Cuando un gestor de paquetes encuentra un lock file, instala exactamente esas versiones exactas, ignorando las versiones disponibles en el registry.

Commitear el lock file al repositorio es una practica esencial de seguridad: garantiza que todos los entornos (desarrollo, CI, staging, produccion) utilizan exactamente las mismas versiones de todas las dependencias. Sin embargo, el lock file solo protege si se verifica su integridad. Si un atacante modifica el lock file para apuntar a una version maliciosa y el diff del commit no se revisa cuidadosamente, el lock file se convierte en el vehiculo del ataque en lugar de la defensa.

### Hashing y verificacion de integridad

Mas alla de la version exacta, los hashes criptograficos proporcionan una capa adicional de verificacion. El archivo `go.sum` de Go, por ejemplo, almacena el hash SHA-256 del modulo descargado; si el contenido descargado no coincide con el hash, Go aborta la operacion y rechaza la dependencia. npm no verifica hashes por defecto en versiones antiguas, pero la inclusion de `integrity` en el `package-lock.json` (generado con Subresource Integrity) permite esta verificacion.

El hashing protege contra la manipulacion del contenido de un paquete en el registry sin cambiar su numero de version (un ataque donde el atacante sube un paquete malicioso reutilizando el nombre de version existente) y contra la corrupcion en la transferencia. Sin embargo, no protege contra la publicacion de una nueva version maliciosa que el lock file no contempla; para eso se necesita la combinacion de lock files + revision de diffs + escaneo de vulnerabilidades.

> La recomendacion practica es combinar multiples defensas: fijar versiones exactas en lock files, verificar hashes de integredad, escanear dependencias con SCA, y mantener un SBOM actualizado que permita respuesta rapida ante CVEs nuevos. Ninguna de estas medidas por si sola es suficiente; su valor reside en la combinacion.

