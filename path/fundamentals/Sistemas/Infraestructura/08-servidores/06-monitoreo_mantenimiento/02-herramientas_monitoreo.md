# Herramientas de monitoreo

Cuando el numero de servidores supera lo que un administrador puede revisar manualmente con comandos, las herramientas de monitoreo se vuelven inevitables. Estas herramientas no solo recolectan metricas de forma automatica, sino que las almacenan, las visualizan y permiten establecer umbrales de alerta. La diferencia entre un entorno gestionado y uno que simplemente "esta corriendo" a menudo radica en la existencia o ausencia de un sistema de monitoreo adecuado.

## Monitoreo local

Las herramientas de monitoreo local estan instaladas directamente en el servidor y ofrecen una vision detallada del estado del sistema sin necesidad de infraestructura externa. Son ideales para servidores aislados, entornos de desarrollo o como complemento a un sistema centralizado.

**Netdata** es una herramienta que genera dashboards web en tiempo real con metricas del sistema, aplicaciones y servicios. Se instala con una unica linea y comienza a recolectar datos inmediatamente, sin configuracion previa. Su modelo de funcionamiento es ligero: un agente por servidor que expone metricas detalladas con una granularidad de un segundo. Es especialmente util para diagnostico rapido en servidores individuales, ya que muestra desde el uso de CPU por nucleo hasta las conexiones de red activas, pasando por metricas de aplicaciones como MySQL, Nginx o Docker.

**Glances** adopta un enfoque diferente: un unico panel de texto que resume el estado completo del sistema. Funciona como una version mejorada de `top` con mas metricas visibles simultaneamente. Admite modo cliente-servidor, lo que permite monitorear varios servidores desde un solo terminal, pero su fortaleza principal es la rapidez con la que permite evaluar el estado de un sistema en una sola pantalla.

```bash
# Netdata: instalacion y ejecucion
curl -Ss https://my-netdata.io/kickstart.sh | sudo bash

# Glances: ejecucion con soporte web
pip install glances
glances -w
```

> Las herramientas locales son excelentes para diagnostico inmediato, pero no resuelven el problema de la visibilidad global: si cada servidor tiene su propio dashboard, el administrador debe revisar uno por uno.

## Monitoreo centralizado

Cuando se gestionan multiples servidores, la centralizacion de metricas es fundamental. Un sistema centralizado recopila datos de todos los servidores en un unico almacen, ofrece una vista unificada y permite establecer alertas globales. La arquitectura tipica consiste en agentes distribuidos que envian metricas a un servidor central, que las almacena y las expone a traves de dashboards.

**Prometheus + Grafana** es la combinacion mas popular en entornos modernos. Prometheus recopila metricas mediante un modelo pull: el servidor central accede periodicamente a cada agente para obtener datos. Esto simplifica la configuracion de red, ya que no es necesario abrir puertos de entrada en cada servidor monitoreado. Grafana se encarga de la visualizacion, ofreciendo dashboards personalizables con soporte para multiples fuentes de datos. La combinacion es especialmente potente en entornos de contenedores y microservicios, donde la infraestructura cambia dinamicamente y la discoverabilidad automatica de metricas es un requisito critico.

**Zabbix** adopta un enfoque mas integral y orientado a entornos empresariales. Incluye agentes, recoleccion de metricas, almacenamiento, visualizacion y alertas en un unico paquete. Su curva de aprendizaje es mayor, pero ofrece funcionalidades avanzadas como descubrimiento automatico de hosts, plantillas predefinidas para miles de dispositivos y un sistema de alertas con escalacion y dependencias. Zabbix funciona con modelo pull o push, y es especialmente robusto en entornos heterogeneos donde coexisten servidores Linux, Windows, switches, routers y otros dispositivos de red.

**Nagios** fue uno de los primeros sistemas de monitoreo y sigue vigente en muchos entornos heredados. Su arquitectura se basa en plugins que verifican el estado de servicios y recursos, devolviendo uno de cuatro estados: OK, WARNING, CRITICAL o UNKNOWN. A diferencia de Prometheus o Zabbix, Nagios no almacena metricas historicamente de forma nativa; su fortaleza esta en la verificacion de estado y las notificaciones. Es una herramienta de verificacion mas que de analisis de tendencias.

## Cuando usar local vs centralizado

La decision entre monitoreo local y centralizado depende del contexto operativo. Un servidor unico con carga de trabajo critica puede beneficiarse de Netdata instalado localmente, ya que la overhead de infraestructura centralizada no esta justificado. En cambio, un cluster de diez servidores requiere un enfoque centralizado para tener visibilidad unificada y correlacionar eventos entre nodos.

En la practica, muchos entornos combinan ambos enfoques: herramientas locales para diagnostico rapido in situ, y un sistema centralizado para la vision global, el historial de metricas y las alertas. La clave es no confundir monitoreo con control: el monitoreo informa, el control actua. Una herramienta de monitoreo que genera cientos de alertas sin un proceso claro de respuesta genera mas ruido que valor.

