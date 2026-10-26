# Modelo IaaS

La computacion en la nube transformo radicalmente la forma en que las organizaciones adquieren y gestionan capacidad de servidor. Antes de IaaS (Infrastructure as a Service), desplegar un servidor nuevo significaba adquirir hardware fisico, rackearlo, cablearlo, configurar la red del centro de datos y esperar semanas o meses antes de tener la maquina operativa. IaaS elimina por completo esa cadena logistica: el proveedor de nube posee, mantiene y opera el hardware — servidores fisicos, switches, routers, refrigeracion, alimentacion electrica — y el cliente interactua con ese hardware exclusivamente a traves de software, pagando unicamente por lo que consume.

IaaS no es simplemente alquilar un servidor remoto. El modelo implica que el proveedor gestiona la capa fisica y la capa de hipervisor, mientras que el cliente asume el control total del sistema operativo, las aplicaciones, los datos y la configuracion de red virtual. Esta division de responsabilidades — conocida como modelo de responsabilidad compartida — es la piedra angular del contrato implicito entre proveedor y cliente. El proveedor garantiza disponibilidad del hardware subyacente; el cliente garantiza que lo que ejecuta sobre ese hardware funciona correctamente.

## Instancias virtuales

Una instancia virtual es la unidad basica de IaaS. Representa una maquina virtual completa con CPU, memoria RAM, almacenamiento y interfaz de red, ejecutada dentro de un hipervisor sobre hardware fisico compartido con otros clientes. Cada proveedor ofrece su propia implementacion y nomenclatura: AWS denomina a sus instancias EC2 (Elastic Compute Cloud), Azure las llama Virtual Machines y Google Cloud Compute Engine. Aunque el nombre cambia, el concepto es identico: el cliente selecciona una configuracion de hardware virtual (numero de vCPU, cantidad de RAM, tipo de almacenamiento) y levanta una instancia en segundos.

Las instancias se clasifican tipicamente por su proposito general — instancias generalizadas con proporcion equilibrada de CPU y memoria — o por optimizacion: instancias computacionales para tareas que demandan procesamiento intensivo (renderizado, simulaciones), instancias de memoria para bases de datos que necesitan grandes volumenes de RAM, o instancias de almacenamiento para cargas de trabajo con requisitos de I/O elevados. El cliente puede elegir la zona geografica donde ejecutar la instancia, lo que afecta latencia, coste y cumplimiento normativo.

## Modelo de pago por uso

El paradigma economico de IaaS elimina el concepto de inversion inicial (CAPEX) en favor de gasto operativo (OPEX). El cliente paga por segundo o por hora de uso real de la instancia, sin compromisos a largo plazo. Cuando la instancia esta apagada, no se cobra por el compute (aunque si por almacenamiento persistente asociado). Este modelo permite escalar horizontalmente durante picos de demanda y reducir a cero fuera de horarios laborales, algo imposible con servidores fisicos donde el hardware esta comprado y instalado independientemente de si se utiliza o no.

> La facturacion por segundo introdujo un nivel de granularidad economica sin precedentes. Un servidor fisico puede costar miles de euros anuales en depreciacion, energia y mantenimiento, incluso si solo se utiliza el 10% de su capacidad. IaaS alinea el coste con el consumo real, eliminando el desperdicio de capacidad ociosa.

## Impacto en la gestion de servidores

IaaS改变了 la relacion entre el equipo de TI y la infraestructura. Antes, un administrador de sistemas era responsable del ciclo completo: desde el cableado de red hasta la instalacion del sistema operativo. Con IaaS, las tareas fisicas desaparecen y se substituyen por llamadas a APIs y configuraciones declarativas. Provisionar un servidor pasa de ser un proceso fisico a ser una operacion de software que puede automatizarse completamente con herramientas como Terraform, CloudFormation o Pulumi.

Esta transicion no elimina la complejidad, la traslada. En lugar de preocuparse por si el disco duro falla fisicamente, el administrador debe preocuparse por si la configuracion de seguridad del grupo de seguridad permite trafico no deseado. En lugar de cablear una red, se diseña una VPC con subnets y route tables. La habilidad tecnica necesaria evoluciona de lo manual-operativo a lo logico-declarativo, lo que exige un cambio de mentalidad significativo en los equipos.

> La verdadera revolucion de IaaS no es tecnica sino organizacional: permite que un equipo de dos personas despliegue una infraestructura que antes requeria un equipo de infraestructura completo. Esto democratizo el acceso a infraestructura de nivel empresarial para startups y proyectos pequenos.

