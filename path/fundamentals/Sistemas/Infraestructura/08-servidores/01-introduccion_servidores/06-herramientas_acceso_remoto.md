# Herramientas de acceso remoto

La administracion de servidores exige acceso remoto como requisito fundamental. Un servidor rara vez se encuentra en la misma sala que el administrador que lo gestiona, y la naturaleza de su operacion — disponibilidad continua, localizacion en datacenters, multiples sedes — hace que la interaccion fisica sea la excepcion. Las herramientas de acceso remoto se clasifican segun la capa del sistema que proporcionan: terminal de texto, interfaz grafica o control completo del hardware.

## SSH: el estandar de facto

SSH (Secure Shell) es el protocolo que domina la administracion remota de servidores Unix/Linux, y su predominio se debe a una combinacion de simplicidad, seguridad y universalidad. Un unico comando `ssh usuario@servidor` establece un canal cifrado punto a punto entre el cliente y el servidor, sobre el cual se transportan comandos, transferencias de archivos y tunnels de red. El cifrado se realiza antes de que ningun dato abandene la maquina local, lo que protege tanto las credenciales como el contenido de la sesion frente a cualquier interlocutor en la red.

El协议o SSH funciona sobre el puerto TCP 22 y utiliza autenticacion por clave publica (asimetrica) o por contrasehna. La autenticacion por clave es preferible en entornos profesionales porque elimina el riesgo de fuerza bruta sobre contrasehnas: se genera un par de claves (publica y privada), la publica se copia al servidor con `ssh-copy-id` y a partir de ese momento la conexion se autentica sin transmitir ningun secreto por la red. La clave privada nunca sale del cliente, y la publica en el servidor no permite derivar la privada.

SSH no se limita a proporcionar un terminal. A traves de la redireccion de puertos, permite acceder a servicios que solo estan disponibles en el servidor o en maquinas detras de el: una base de datos con un puerto cerrado al exterior puede exponerse temporalmente a traves de un tunnel SSH para su administracion. SCP y SFTP, que operan sobre el mismo protocolo, permiten transferir archivos con cifrado completo. La combinacion de estas capacidades en un unico protocolo es una de las razones por las que SSH se ha convertido en la herramienta esencial del administrador de sistemas.

La razon por la que SSH desplazo a Telnet y rlogin como protocolos de acceso remoto es su manejo del cifrado. Telnet transporta todo el contenido — incluidas las contrasehnas — en texto plano, lo que lo hace inseguro en cualquier red que no sea de confianza total. SSH resolvio este problema incorporando cifrado integral desde su diseno, y lo hizo con un modelo de autenticacion por clave que resolvia simultaneamente el problema de la verificacion de identidad sin transmitir secretos. Aunque existen alternativas mas modernas, SSH se mantiene como estandar porque su modelo es correcto, su implementacion es madura y su presencia en praticamente todos los sistemas operativos lo convierte en el menor comun denominador de la administracion remota.

## Consolas web en la nube

Los proveedores de servicios en la nube ofrecen consolas web que permiten gestionar las maquinas virtuales desde un navegador. AWS, Azure y Google Cloud proporcionan terminales web integrados que ejecutan comandos sobre la instancia sin necesidad de abrir un cliente SSH local. Estas consolas operan a traves de la API del proveedor y se comunican con un agente instalado en la maquina virtual, lo que las convierte en un recurso de emergencia cuando la conectividad SSH se pierde.

Las consolas web son especialmente utiles durante la fase de aprovisionamiento, cuando la maquina virtual aun no tiene las claves SSH configuradas o el firewall bloquea el puerto 22. Tambien resultan practicas para acceder a maquinas cuya tabla de rutas se ha corrompido o cuya interfaz de red ha dejado de funcionar, ya que operan sobre un canal de red independiente al del sistema operativo. Sin embargo, su uso habitual tiene desventajas: la latencia de la respuesta es mayor que en una sesion SSH directa, la integracion con herramientas de automatizacion es limitada, y la experiencia de uso en sesiones prolongadas es inferior.

La consola web no sustituye a SSH en la administracion diaria; la complementa como mecanismo de acceso de emergencia y como herramienta de provision inicial. En entornos de produccion, lo habitual es que la consola web este disponible pero que rara vez se utilize, reservandose para situaciones en las que el acceso SSH normal no es posible.

## IPMI, iLO e iDRAC: acceso a nivel de hardware

Estos sistemas de gestion remota operan en una capa inferior al sistema operativo: son procesadores dedicados (BMC, Baseboard Management Controller) integrados en la placa base del servidor que funcionan con energia propia y una tarjeta de red dedicada, independiente de la red del sistema operativo. IPMI (Intelligent Platform Management Interface) es el estandar, mientras que iLO (HP) e iDRAC (Dell) son implementaciones propietarias que lo extienden con funcionalidades adicionales.

El acceso a estas interfaces se realiza a traves de un navegador web que se conecta a la IP del BMC, normalmente en una subred de gestion separada. Desde el navegador se puede ver la consola de video del servidor (incluso antes de que el sistema operativo arranque), encender o apagar el equipo, montar imagenes ISO de instalacion de forma remota y monitorizar metricas hardware como temperatura, voltajes y velocidades de ventiladores. Estas capacidades son equivalentes a estar fisicamente frente al servidor con teclado, monitor y mano en el boton de encendido.

La principal ventaja de IPMI/iLO/iDRAC es que operan fuera del sistema operativo, lo que los hace inmunes a cualquier fallo del software. Si el sistema operativo se bloquea, si el kernel panicca o si un actualizacion deja el sistema inoperable, el BMC sigue funcionando y permite recuperar el equipo. Este nivel de acceso es indispensable en servidores remotos o en datacenters donde el desplazamiento fisico tiene un coste significativo. El inconveniente es que las interfaces BMC han sido historicamente un vector de vulnerabilidades de seguridad, ya que su software integrado a menudo no se actualiza con la misma frecuencia que el sistema operativo principal.

## Comparativa y uso combinado

Cada herramienta de acceso responde a una necesidad diferente y las tres coexisten en una infraestructura bien gestionada. SSH es la herramienta de uso diario para la administracion del sistema operativo y las aplicaciones. La consola web del proveedor cloud sirve como respaldo cuando SSH no esta disponible y como punto de entrada inicial. IPMI/iLO/iDRAC proporciona control absoluto sobre el hardware y permite recuperar el servidor desde situaciones que las otras dos herramientas no pueden resolver.

