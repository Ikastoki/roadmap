# Modelos de implementacion

La forma en que se despliega un servidor tiene implicaciones directas sobre costes, escalabilidad, mantenimiento y control. A lo largo de las ultimas decadas, el modelo de implementacion ha evolucionado desde la compra y mantenimiento de equipamiento fisico propio hasta los servicios gestionados en la nube, pasando por la virtualizacion como punto intermedio. Cada modelo responde a necesidades distintas y presenta compromisos que deben evaluarse en funcion de cada escenario.

## On-premises

El modelo on-premises implica adquirir el hardware, instalarlo en un datacenter o sala de servidores propio y asumir toda la responsabilidad sobre su funcionamiento. Esto incluye la infraestructura fisica — racks, cableado, climatizacion, alimentacion con UPS y generador — asi como la gestion del sistema operativo, las actualizaciones de seguridad, las copias de seguridad y la monitorizacion continua.

La ventaja principal es el control absoluto: se decide que hardware se instala, que software se ejecuta, como se configura la red y que politicas de seguridad se aplican. Para organizaciones con requisitos regulatorios estrictos, datos sensibles o baja tolerancia a la dependencia de terceros, este modelo sigue siendo la referencia. Sin embargo, los costes iniciales son elevados: la compra de un servidor de gama media con discos redundantes, memoria ECC y fuentes duplicadas puede superar los 10.000 euros, a los que se suman los gastos recurrentes de electricidad, refrigeracion y personal de mantenimiento.

La escalabilidad en on-premises es discrecional pero lenta. Ampliar la capacidad implica adquirir nuevo hardware, esperar su entrega, instalarlo fisicamente y configurarlo, un proceso que puede tardar semanas o meses. Este modelo penaliza la incertidumbre en la demanda: si se compra hardware para el pico de carga y ese pico solo se alcanza dos veces al ano, la mayoria del tiempo el equipamiento permanece subutilizado.

## Virtualizado

La virtualizacion introduce una capa de abstraccion — el hipervisor — que permite ejecutar multiples sistemas operativos sobre el mismo hardware fisico. VMware vSphere, Proxmox VE y KVM son las soluciones mas extendidas, y cada una ofrece un equilibrio distinto entre funcionalidad, coste y comunidad. El hipervisor gestiona los recursos del host (CPU, memoria, disco, red) y los distribuye entre las maquinas virtuales segun la configuracion definida por el administrador.

Este modelo mejora significativamente la utilizacion del hardware. Un servidor fisico con 64 GB de RAM y 16 nucleos puede ejecutar simultaneamente ocho maquinas virtuales con 8 GB y 2 nucleos cada una, en lugar de tener un unico sistema operativo que deje recursos sin utilizar la mayor parte del tiempo. La virtualizacion tambien facilita funcionalidades como las snapshots — instantaneas del estado de una maquina virtual en un momento dado — que permiten revertir cambios problematicos en segundos.

La movilidad es otra ventaja clave. Una maquina virtual se puede migrar en caliente entre dos hosts fisicos sin interrupcion del servicio, lo que permite realizar mantenimiento hardware en un servidor sin apagar las cargas de trabajo que aloja. Sin embargo, la virtualizacion introduce una capa adicional de complejidad y un punto de fallo potencial: si el hipervisor cae, todas las maquinas virtuales que aloja se apagan simultaneamente, lo que exige configuraciones de alta disponibilidad con clustering y almacenamiento compartido.

## Cloud computing

El modelo en la nube delega la gestion del hardware y, en muchos casos, del sistema operativo a un proveedor externo como AWS, Microsoft Azure o Google Cloud. La propuesta de valor es clara: pagar por lo que se utiliza, sin compromisos iniciales y con la capacidad de escalar automaticamente segun la demanda. Las variantes mas comunes son IaaS (Infrastructure as a Service), donde se alquila infraestructura virtual completa, y PaaS (Platform as a Service), donde el proveedor gestiona tanto la infraestructura como el entorno de ejecucion, permitiendo al desarrollador desplegar codigo sin preocuparse por el sistema operativo ni el hardware subyacente.

IaaS ofrece la mayor flexibilidad: se controla el sistema operativo, las aplicaciones y la configuracion de red, pero se delega la fisica, la climatizacion y la redundancia de hardware. PaaS va mas alla al abstraer tambien el sistema operativo, lo que reduce la carga operativa pero limita el control sobre la configuracion del entorno. Existe ademas SaaS (Software as a Service), donde el servicio completo — incluyendo la aplicacion — esta gestionado por el proveedor, como es el caso de Google Workspace o Microsoft 365.

La nube no es inherentemente mejor ni peor que los modelos anteriores; simplemente desplaza los costes de capital (CAPEX) hacia costes operativos (OPEX) y traslada parte de la responsabilidad de mantenimiento al proveedor. Los compromisos incluyen la dependencia del proveedor (vendor lock-in), la dificultad de estimar costes a largo plazo cuando el consumo es variable, y las preocupaciones sobre residencia de datos y soberania. Para cargas de trabajo con picos impredecibles, entornos de desarrollo o proyectos con ciclo de vida corto, la nube ofrece una agilidad que los modelos locales no pueden igualar.

## Tendencia hibrida

La mayoria de organizaciones no elige un modelo exclusivo, sino que combina on-premises, virtualizacion y nube en una estrategia hibrida. Los servicios criticos con requisitos de latencia baja o regulacion estricta se mantienen en infraestructura propia, mientras que las cargas de trabajo variables o los entornos de prueba se migran a la nube. Esta aproximacion requiere herramientas de orquestacion y monitorizacion que operen a traves de los distintos entornos, como Terraform para la gestion de infraestructura como codigo o Prometheus para la observabilidad unificada.

