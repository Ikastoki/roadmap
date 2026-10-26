# Red y seguridad en la nube

La infraestructura de red en la nube es un sistema de capas superpuestas que abstrae la complejidad del hardware fisico (switches, routers, firewalls fisicos, balanceadores) en componentes virtuales configurables desde software. Comprender estas capas es esencial porque los errores de configuracion de red en la nube son la causa mas comun de brechas de seguridad: un grupo de seguridad mal configurado o una subnet publica expuesta sin querer pueden convertir un servidor interno en un objetivo accesible desde Internet en cuestion de segundos.

## Grupos de seguridad

Los grupos de seguridad funcionan como firewalls stateful a nivel de instancia. Cada regla define una combinacion de protocolo, puerto y origen/destino permitido. A diferencia de los firewalls tradicionales donde la configuracion puede ser compleja y distribuida, los grupos de seguridad en la nube son simples, granulares y se aplican inmediatamente. El trafico que no coincide con ninguna regla permisiva se bloquea por defecto, lo que implementa el principio de minimo privilegio de forma nativa.

> Un error comun es configurar reglas de entrada abiertas (0.0.0.0/0 en todos los puertos) durante la fase de pruebas y olvidarlas al pasar a produccion. Los grupos de seguridad deben ser tratados como codigo: revisados, versionados y auditados regularmente.

## VPC y subnets

Una VPC (Virtual Private Cloud) es una red virtual aislada dentro de la cuenta del cliente donde se despliegan las instancias. Dentro de una VPC se definen subnets — segmentos de la red con rangos de direcciones IP propios — que pueden ser publicas (con acceso directo a Internet a traves de una Internet Gateway) o privadas (sin acceso directo, comunicandose con el exterior unicamente a traves de NAT Gateways o proxies).

La separacion entre subnets publicas y privadas es un patron arquitectonico fundamental. Las bases de datos, servidores de aplicacion internos y servicios de backend se despliegan en subnets privadas, inaccesibles desde Internet directamente. Los servidores frontales, balanceadores de carga y APIs publicas habitan en subnets publicas. Esta segmentacion reduce la superficie de ataque significativamente: aunque un servidor en subnet privada sea comprometido, el atacante no puede acceder a el directamente desde Internet.

## Modelo de responsabilidad compartida

El modelo de responsabilidad compartida es el contrato implicito que define que gestiona el proveedor de nube y que gestiona el cliente. En terminos de seguridad, el proveedor es responsable de la seguridad "del" cloud: seguridad fisica de los centros de datos, del hipervisor, de la infraestructura de red global, de la proteccion contra ataques DDoS a nivel de plataforma y de la infraestructura de almacenamiento encriptada.

El cliente es responsable de la seguridad "en" la nube: configuracion de grupos de seguridad, gestion de identidades y accesos (IAM), encriptacion de datos en reposo y en transito, parcheo del sistema operativo de las instancias, proteccion de las claves de acceso y la configuracion de la red virtual. Esta division es critica porque muchos clientes asumen incorrectamente que el proveedor protege todo, cuando en realidad las capas superiores — las que contienen los datos y las aplicaciones — son responsabilidad exclusiva del cliente.

> El modelo de responsabilidad compartida varia segun el servicio utilizado. En IaaS, el cliente asume la mayor parte de la responsabilidad (SO, aplicaciones, datos). En PaaS (como AWS Lambda o Azure App Service), el proveedor gestiona tambien el runtime y el parcheo del SO. En SaaS (como Gmail o Salesforce), la responsabilidad del cliente se reduce casi exclusivamente a la gestion de datos y accesos.

