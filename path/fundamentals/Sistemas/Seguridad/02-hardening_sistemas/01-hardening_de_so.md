# Hardening de Sistemas Operativos

El hardening de sistemas operativos es el proceso de reducir la superficie de ataque de un sistema eliminando servicios innecesarios, aplicando parches de seguridad, configurando permisos restrictivos y deshabilitando funciones que no son requeridas para el normal funcionamiento del servidor. Este proceso es fundamental porque el sistema operativo constituye la base sobre la cual se ejecutan todas las demas capas de infraestructura; una configuracion insegura en este nivel compromete la seguridad de todas las aplicaciones y datos alojados en el equipo.

La importancia del hardening radica en que los sistemas operativos vienen con una configuracion por defecto orientada a la funcionalidad y compatibilidad, no a la seguridad. Servicios como FTP anónimo, puertos abiertos innecesarios o usuarios con privilegios excesivos son comunes en instalaciones frescas. Cada uno de estos elementos representa un vector de ataque potencial que un adversario podria explotar. El proceso de hardening transforma un sistema generico en uno especializado y resistente, donde cada componente ha sido evaluado y configurado exclusivamente para el proposito asignado.

## Gestion de Cuentas y Autenticacion

La primera linea de defensa en cualquier sistema operativo comienza con el control de acceso a traves de cuentas de usuario. Los sistemas vienen con multiples cuentas preconfiguradas que deben ser revisadas y, en muchos casos, deshabilitadas. La cuenta de root o Administrator debe utilizarse exclusivamente para tareas administrativas criticas, nunca para operaciones cotidianas. Se recomienda crear cuentas individuales con privilegios limitados para cada administrador y auditar periodicamente el listado de cuentas activas.

La politica de contrasenas es otro componente esencial. Los sistemas operativos permiten configurar requisitos de complejidad, longitud minima, historial de contrasenas y fechas de expiracion. Estas politicas deben implementarse obligatoriamente a traves de PAM (Pluggable Authentication Modules) en Linux o Group Policy en Windows. Adicionalmente, se debe deshabilitar el acceso directo a la cuenta de root y obligar la elevacion de privilegios mediante sudo o runas.

## Control de Servicios y Puertos

Un servicio corriendo en un servidor es un potencial punto de compromiso. El principio basico establece que todo servicio que no sea estrictamente necesario debe ser deshabilitado. Esto incluye servicios de red como FTP, Telnet, rsh, y servicios de impresion en servidores que no manejan documentos. En Linux, herramientas como systemctl permiten identificar y deshabilitar servicios innecesarios. En Windows, el Gestor de Servicios proporciona funcionalidad similar.

La verificacion de puertos abiertos complementa el control de servicios. Herramientas como netstat, ss o lsof revelan que procesos estan escuchando en cuales puertos. Cada puerto abierto debe justificarse con un proposito documentado. Los puertos que no estan explicitamente autorizados deben ser bloqueados a traves del firewall del sistema. Esta practica reduce significativamente la superficie de ataque disponible para un escaneo de red.

## Aplicacion de Parches y Actualizaciones

Los sistemas operativos y las aplicaciones instaladas contienen vulnerabilidades que son descubiertas continuamente por investigadores de seguridad y, potencialmente, por actores maliciosos. La gestion de parches es el proceso de identificar, evaluar e instalar actualizaciones de seguridad de manera oportuna. Los proveedores de sistemas operativos publican actualizaciones regulares que corrigen vulnerabilidades criticas; ignorar estas actualizaciones es exponer el sistema a exploits conocidos y facilmente automatizados.

La estrategia de parchado debe balancear estabilidad y seguridad. Los parches criticos de seguridad deben aplicarse lo antes posible, idealmente dentro de los primeros 30 dias de su publicacion. Los parches no criticos pueden programarse en ciclos de mantenimiento regulares. Antes de aplicar parches en entornos productivos, se recomienda probarlos en un entorno de staging para verificar compatibilidad con las aplicaciones existentes.

## Configuracion de Auditoria y Registro

La auditoria del sistema operativo permite detectar actividades sospechosas, investigar incidentes y demostrar cumplimiento normativo. Todos los sistemas operativos modernos incluyen mecanismos de registro que capturan eventos de inicio de sesion, cambios en archivos criticos, acceso a recursos y ejecucion de comandos. Estos registros deben ser configurados para capturar la informacion relevante y almacenarse de forma segura en un servidor centralizado de logs.

La centralizacion de registros es critica porque un atacante comprometeria localmente el sistema y podria manipular o eliminar los logs para cubrir sus huellas. Herramientas como rsyslog, journald en Linux o el Visor de Eventos en Windows permiten enviar registros a un servidor remoto de forma cifrada. La retencion de logs debe seguir las politicas de la organizacion, generalmente entre 90 dias y un ano dependiendo de los requisitos de cumplimiento.

## Aislamiento y Segmentacion

El aislamiento del sistema operativo busca limitar el impacto de una posible compromision. Esto se logra implementando SELinux o AppArmor en Linux para restringir las acciones que cada proceso puede realizar, utilizando contenedores o maquinas virtuales para separar funcionalidades, y configurando permisos de archivos e directorios de manera restrictiva. El principio de minimo privilegio se aplica a nivel de proceso: cada servicio debe ejecutarse con los permisos minimos necesarios para funcionar.

La segmentacion a nivel de sistema operativo incluye la creacion de particiones separadas para directorios criticos como /var, /tmp y /home, con opciones de montaje que previenen la ejecucion de binarios en temporales y limitan el uso de recursos. Estas medidas aseguran que la compromision de un componente no facilite la escalada de privilegios o el movimiento lateral hacia otros subsistemas.

## Configuracion de Red a Nivel de SO

El sistema operativo ofrece controles de red adicionales alla del firewall dedicado. La deshabilitacion del enrutamiento IP, la proteccion contra spoofing a traves de filtros de ruta inversa, y la configuracion de parametros TCP/IP como la proteccion contra ataques SYN flood son configuraciones que se aplican directamente en el nivel del sistema operativo. Herramientas como sysctl en Linux permiten ajustar estos parametros de forma persistente.

La configuracion de la pila de red del sistema operativo tambien incluye la deshabilitacion de protocolos inseguros como Telnet y FTP en favor de alternativas cifradas como SSH y SFTP. La version de IP utilizada, las opciones de TCP y los buffers de red deben configurarse considerando tanto el rendimiento como la seguridad del sistema.

## Automatizacion del Hardening

El hardening manual es propenso a errores y dificil de mantener a escala. Las herramientas de automatizacion como Ansible, Chef o Puppet permiten definir configuraciones de seguridad como codigo, garantizando consistencia entre servidores y facilitando la auditoria de conformidad. CIS Benchmarks y DISA STIGs proporcionan guias de configuracion estandarizadas que pueden implementarse automaticamente.

La automatizacion tambien permite detectar desviaciones de la configuracion segura establecida. Los sistemas de monitoreo de conformidad pueden escanear periodicamente los servidores y alertar cuando una configuracion se desvia del estandar. Esta capacidad es esencial para mantener la postura de seguridad a lo largo del tiempo, especialmente en entornos con multiples servidores que deben mantenerse homogeneos.

---

