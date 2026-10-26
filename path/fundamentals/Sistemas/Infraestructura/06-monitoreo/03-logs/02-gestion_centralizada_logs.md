# Gestion Centralizada de Logs

En un entorno monolitico con un solo servidor, los logs pueden leerse directamente con `tail -f /var/log/syslog` y la respuesta a un incidente comienza abriendo una terminal. Pero cuando una peticion HTTP atraviesa un balanceador de carga, un API gateway, tres microservicios, dos colas de mensajes y una base de datos, cada uno de esos componentes genera sus propios logs en sus propios servidores. El operador se enfrenta entonces a un problema de escala: no puede hacer SSH a cada maquina individualmente para reconstruir el historial de una sola peticion. La gestion centralizada de logs nace de esta necesidad: reunir toda la informacion de registro en un lugar comun donde sea posible buscarla, cruzarla y analizarla de forma unificada.

## El problema de los logs distribuidos

Cada servicio en una arquitectura de microservicios escribe sus logs en su propio sistema de archivos o su propio contenedor. Kubernetes, por ejemplo, ejecuta cada pod con su propio flujo de salida de logs. Si un error se manifiesta en el servicio de pedidos pero su causa real esta en el servicio de inventario, el operador necesita acceder a los logs de ambos servicios simultaneamente para reconstruir la cadena de eventos. Sin centralizacion, esto implica abrir multiples terminales SSH, copiar y pegar timestamps, y reconstruir manualmente la secuencia. Este proceso es lento, propenso a errores y completamente inviable durante un incidente con presion de tiempo.

La centralizacion tambien resuelve problemas de persistencia. Los logs en disco local se pierden si un contenedor se reinicia, un nodo falla o un pod se reprograma en otro maquina. Un sistema centralizado captura los logs antes de que desaparezcan del origen, garantizando que la evidencia diagnostica este disponible incluso despues de que el componente que la genero ya no exista.

## Pipeline de gestion de logs

El proceso de gestion centralizada de logs sigue un flujo estructurado en cuatro fases que se encadenan de forma continua.

### Recoleccion

Cada fuente de logs (servidores, contenedores, aplicaciones, firewalls, balanceadores) debe tener un agente o mecanismo que capture su salida. En contenedores Docker y Kubernetes, esto se logra normalmente redirigiendo stdout/stderr a un daemon de logs. En servidores tradicionales, agentes como Filebeat, Fluentd o Vector leen los archivos de log del disco y envian las entradas nuevas a medida que aparecen. La recoleccion debe ser lo menos intrusiva posible: el agente no debe consumir recursos significativos ni modificar el formato del log original.

### Transporte

Una vez recolectados, los logs deben viajar hacia el sistema centralizado. Este transporte se realiza generalmente a traves de protocolos como TCP o UDP, y en entornos mas grandes se utilizan colas de mensajes como Kafka o NATS para absorber picos de volumen y proporcionar un buffer de seguridad. Si el sistema de almacenamiento temporalmente no esta disponible, la cola retiene los logs hasta que se restablezca la conexion, evitando perdida de datos. El transporte debe incluir mecanismos de compresion y, preferiblemente, cifrado para proteger la informacion sensible que los logs pueden contener.

### Almacenamiento

El almacenamiento es la fase donde los logs persisten y se organizan para su consulta. Los sistemas de almacenamiento varian segun la necesidad: Elasticsearch ofrece capacidades de indexado avanzado y busqueda full-text, Loki almacena logs indexados unicamente por sus etiquetas (reduciendo coste de almacenamiento), y soluciones como S3 o GCS proporcionan almacenamiento a largo plazo a bajo coste. La decision de almacenamiento influye directamente en el coste y la velocidad de consulta: indexar cada campo de un log JSON es mas costoso en espacio pero mas rapido en busqueda, mientras que indexar solo por etiquetas es mas economico pero limita las consultas.

### Consulta e Indexacion

Los logs almacenados deben ser consultables de forma eficiente. Un buen sistema de gestion centralizada proporciona una interfaz donde el operador puede buscar por timestamp, nivel de severidad, servicio, usuario, mensaje o cualquier campo estructurado. La indexacion es lo que hace esto posible: cada campo indexado se convierte en un indice que permite busquedas en milisegundos, incluso con millones de registros. Sin indexacion, cada consulta requeriria un escaneo completo del almacenamiento, lo cual es impracticable en produccion.

## Retencion y Archivado

No todos los logs tienen el mismo valor a largo plazo. Los logs de TRACE y DEBUG de hace seis meses rara vez tienen utilidad operativa, pero los logs de ERROR y FATAL de auditoria pueden ser requeridos por regulacion durante anos. La politica de retencion establece cuanto tiempo se mantienen los logs accesibles en el sistema de consulta activa, y que sucede despues de ese periodo.

Un esquema tipico de retencion mantiene los logs completos (todos los niveles) durante 7 a 30 dias, los logs de nivel INFO y superior durante 90 dias, y los logs de ERROR y FATAL durante un periodo de uno a tres anos, dependiendo de los requisitos de cumplimiento normativo. Los logs que superan estos periodos se archivan en almacenamiento de bajo coste (como S3 Glacier o Azure Archive) donde permanecen accesibles pero no indexados para consulta rapida.

> Las politicas de retencion no son solo una decision tecnica: tienen implicaciones legales y de compliance. Sectores como finanzas, salud o gobierno pueden requerir retenciones minimas de logs de auditoria durante periodos de cinco a siete anos. Definir estas politicas antes de implementar el sistema evita sorpresas regulatorias.

