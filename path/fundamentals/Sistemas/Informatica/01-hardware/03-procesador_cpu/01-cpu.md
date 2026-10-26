# Qué es la CPU

La unidad central de procesamiento (*Central Processing Unit*) es el componente computacional principal de una computadora. Es el cerebro del sistema: ejecuta las instrucciones de los programas, realiza cálculos aritméticos y lógicos, y coordina las operaciones de todos los demás componentes. Cada acción que un ordenador realiza —desde dibujar un pixel en la pantalla hasta calcular la trayectoria de un cohete— pasa por la CPU en algún momento.

La CPU no es solo un chip rápido: es un sistema complejo de millones o miles de millones de transistores organizados en circuitos especializados que cooperan para procesar información a velocidades extraordinarias. Comprender cómo funciona es fundamental para entender por qué un sistema determinado tiene cierto rendimiento y dónde están sus limitaciones.

## Función principal

La CPU ejecuta un ciclo continuo que consta de cuatro fases fundamentales: búsqueda (*fetch*), decodificación (*decode*), ejecución (*execute*) y escritura (*writeback*). En cada ciclo, la CPU lee una instrucción de la memoria, interpreta qué operación debe realizar, ejecuta esa operación sobre los datos y almacena el resultado.

Este ciclo se repite miles de millones de veces por segundo. Un procesador a 4 GHz completa 4.000 millones de ciclos por segundo, y en cada ciclo puede ejecutar una o más operaciones dependiendo de la arquitectura. La velocidad a la que la CPU puede completar este ciclo determina fundamentalmente el rendimiento del sistema.

La CPU también gestiona las interrupciones: señales de los dispositivos de E/S que requieren atención inmediata. Cuando un usuario pulsa una tecla o se completa una transferencia de datos por red, la CPU interrumpe su tarea actual para atender la petición, y luego reanuda la tarea anterior. Esta capacidad de gestionar múltiples tareas de forma concurrente es esencial para el funcionamiento de sistemas multitarea.

## Evolución histórica

La evolución de las CPU puede dividirse en eras tecnológicas claras:

**Era de los tubos de vacío (1940-1950):** las primeras computadoras electrónicas como la ENIAC utilizaban tubos de vacío como elementos de conmutación. La ENIAC contenía 17.468 tubos, pesaba 30 toneladas y consumía 150 kW. A pesar de su enormidad, su capacidad de cálculo era inferior a la de una calculadora moderna.

**Era de los transistores (1950-1960):** la invención del transistor en 1947 por Bell Labs reemplazó los tubos de vacío por componentes sólidos, reduciendo drásticamente el tamaño, el consumo y el calor. Los ordenadores de esta época, como el IBM 7090, fueron utilizados para aplicaciones científicas y militares.

**Era de los circuitos integrados (1960-1970):** los circuitos integrados permitieron colocar múltiples transistores en un solo chip de silicio. Los procesadores de esta época, como el Intel 4004 (1971), el primer microprocesador comercial, contenían apenas unos pocos miles de transistores.

**Era del microprocesador (1970-presente):** desde el Intel 8080 (1974) hasta los procesadores actuales con miles de millones de transistores, esta era ha visto una aceleración sin precedentes en el poder de cálculo. Los procesadores modernos combinan múltiples núcleos, caches multinivel, unidades gráficas integradas y controladores de memoria en un solo chip.

> La Ley de Moore, que predijo la duplicación del número de transistores cada dos años, se cumplió durante casi medio siglo. En 1971, el Intel 4004 contenía 2.300 transistores. Los procesadores actuales superan los 100.000 millones.

## Tipos de CPU

Las CPU se clasifican según su propósito, rendimiento y factor de forma:

- **CPU de escritorio:** diseñadas para un equilibrio entre rendimiento y consumo energético. Son las más comunes en ordenadores personales.
- **CPU de portátil:** optimizadas para bajo consumo y menor generación de calor, ya que operan en espacios cerrados con refrigeración limitada.
- **CPU de servidor:** priorizan el número de núcleos, la capacidad de memoria y la fiabilidad sobre la velocidad single-core. Soportan múltiples canales de memoria y corrección de errores ECC.
- **CPU embebidas:** diseñadas para sistemas especializados como routers, dispositivos IoT o sistemas de control industrial. Priorizan la eficiencia energética y la fiabilidad sobre el rendimiento bruto.

## CPU integradas

Las CPU integradas o *APU* (*Accelerated Processing Unit*) combinan el procesador central con una unidad gráfica integrada en el mismo chip. Esta integración elimina la necesidad de una tarjeta gráfica dedicada para tareas básicas, reduciendo el coste, el consumo y la complejidad del sistema.

### CPUs modernas

Las CPU integradas actuales han alcanzado un nivel de rendimiento gráfico que las hace viables para gaming casual, edición de video básica y productividad. Los AMD Ryzen con gráficos Radeon integrados y los Intel Core con Iris Xe representan lo último en esta tecnología.

### CPUs de escritorio

Las CPU de escritorio integradas son populares en sistemas donde el espacio, el coste o el consumo son preocupaciones principales. Un APU AMD Ryzen 7 8700G, por ejemplo, ofrece suficiente potencia gráfica para jugar títulos populares a 1080p sin necesidad de tarjeta gráfica dedicada.

### CPUs móviles

En el mercado móvil, las CPU integradas son la norma. Los Apple Silicon (M1, M2, M3, M4) y los Qualcomm Snapdragon X representan la convergencia entre procesadores de alta eficiencia y gráficos integrados potentes, demostrando que las CPU integradas pueden competir con soluciones dedicadas en muchos escenarios.

### CPUs para servidores

Las CPU de servidor integradas están orientadas a cargas de trabajo que no requieren gráficos dedicados, como servidores web, bases de datos o máquinas virtuales. Los AMD EPYC e Intel Xeon ofrecen entre 16 y 128 núcleos, grandes capacidades de memoria y soporte ECC, maximizando el número de tareas concurrentes por watt.

## Integración con la placa base y memoria

La CPU moderna no opera de forma aislada. Sus controladores integrados determinan cómo se comunica con la placa base y la memoria:

**Controladores de memoria integrados:** los procesadores modernos incluyen el controlador de memoria directamente en el chip, eliminando el Northbridge como componente separado. Esto reduce la latencia de acceso a la memoria y mejora el rendimiento.

**Carriles PCIe integrados:** la CPU proporciona los carriles PCIe de alta velocidad para la GPU y los SSD NVMe. El número de carriles disponibles varía entre 16 y 28 según el modelo, y determina cuántos dispositivos de alta velocidad pueden conectarse sin compartir ancho de banda.

**Controlador de E/S:** algunos procesadores incluyen controladores USB, Ethernet e incluso gráficos integrados, reduciendo la dependencia del chipset para dispositivos de menor velocidad.

> La integración de controladores en la CPU ha simplificado el diseño de las placas base pero ha transferido complejidad al procesador. Un fallo en el controlador de memoria o en los carriles PCIe de la CPU puede inutilizar funcionalidades críticas de la placa base, y a diferencia de un chip externo, no es reemplazable.

