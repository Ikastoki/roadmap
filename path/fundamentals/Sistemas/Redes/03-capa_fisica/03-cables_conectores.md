# Tipos de cables y conectores

La seleccion de cables y conectores es una decision practica que afecta directamente el rendimiento, la fiabilidad y la escalabilidad de una red cableada. Cada tipo de cable tiene caracteristicas electricas y mecanicas que lo hacen mas adecuado para ciertos usos: corta distancia, larga distancia, alta velocidad, baja interferencia, etc.

Los estandares de cableado (TIA/EIA-568) definen las especificaciones tecnicas que deben cumplir los cables y conectores para garantizar la interoperabilidad y el rendimiento. Conocer estas especificaciones es esencial para disenar, instalar y mantener infraestructura de red profesional.

## Par trenzado: UTP y STP

**UTP (*Unshielded Twisted Pair*):** cable de cobre con pares trenzados sin blindaje. Es el medio mas comun en redes LAN. Cada par de hilos esta trenzado para reducir la interferencia electromagneticas entre pares (*crosstalk*). Las categorias definen el rendimiento maximo:

- **Cat5e:** soporta hasta 1 Gbps a 100 metros. La minima aceptable para redes modernas.
- **Cat6:** soporta hasta 10 Gbps a 55 metros (o 1 Gbps a 100 metros). Mejor proteccion contra crosstalk que Cat5e.
- **Cat6a:** soporta 10 Gbps a 100 metros. Estandar recomendado para nuevas instalaciones.
- **Cat7:** soporta 10 Gbps con blindaje individual de pares y blindaje global. Conector Tera o GG45, no es compatible con RJ45 estandar.
- **Cat8:** soporta 25/40 Gbps a 30 metros. Para corta distancia en centros de datos.

**STP (*Shielded Twisted Pair*):** incluye un blindaje metalico alrededor de los pares o del cable completo, proporcionando mayor proteccion contra interferencias electromagneticas. Es necesario en entornos industriales con maquinaria pesada o en instalaciones cercanas a fuentes de interferencia.

## Coaxial

El cable coaxial consiste en un conductor central rodeado de aislante, una malla conductora y una cubierta protectora. Ofrece mejor proteccion contra interferencias que el par trenzado pero es mas rigido y caro.

- **RG-6:** utilizado para television por cable y internet por cable (DOCSIS). Impedancia de 75 ohmios.
- **RG-59:** utilizado para television analogica y CCTV. Impedancia de 75 ohmios pero menor calidad que RG-6.

El coaxial fue el estandar en redes Ethernet tempranas (10BASE2 con BNC, 10BASE5 con AUI) pero ha sido completamente reemplazado por par trenzado en LAN modernas.

## Fibra optica

La fibra optica transmite datos como pulsos de luz a traves de un nucleo de vidrio o plastico ultrafino. Ofrece ancho de banda extremadamente alto, inmunidad total a interferencias electromagneticas y largas distancias.

**Fibra monomodo (*Single-Mode*, SMF):** un solo nucleo de vidrio muy fino (8-10 micrometros) por donde viaja un unico rayo de luz. Ofrece la mayor distancia (hasta 100 km o mas) y la mayor capacidad. Util en WAN, backbone de centros de datos y conexiones entre edificios.

**Fibra multimodo (*Multi-Mode*, MMF):** nucleo mas grueso (50-62.5 micrometros) por donde viajan multiples rayos de luz simultaneamente. Distancia menor (hasta 2 km) pero los equipos de transmision (LED, VCSEL) son mas economicos. Util en LAN y dentro de centros de datos.

## Conectores

**RJ45 (*Registered Jack 45*):** conector estandar para cables de par trenzado Ethernet. Tiene 8 pines (8P8C) y es el conector mas comun en redes LAN.

**BNC (*Bayonet Neill-Concelman*):** conector de encaje por bayoneta utilizado en cables coaxiales. Se usaba en Ethernet 10BASE2 y actualmente en CCTV y equipamiento de RF.

**Conectores de fibra optica:**

- **SC (*Subscriber Connector*):** conector de encaje cuadrado, facil de conectar y desconectar. Util en fibra monomodo y multimodo.
- **LC (*Lucent Connector*):** conector pequeno y compacto, el mas comun en centros de datos modernos. Duplica la densidad de puertos frente a SC.
- **ST (*Straight Tip*):** conector de encaje por bayoneta, similar a BNC. Comun en instalaciones antiguas pero reemplazado por LC y SC.

## Estandares de cableado: T568A y T568B

Los estandares TIA/EIA-568 definen el orden de los hilos en el conector RJ45. Existen dos variantes principales:

**T568A:** orden de colores: blanco-verde, verde, blanco-naranja, azul, blanco-azul, naranja, blanco-marron, marron.

**T568B:** orden de colores: blanco-naranja, naranja, blanco-verde, azul, blanco-azul, verde, blanco-marron, marron.

La diferencia entre ambos es el intercambio de los pares verde y naranja. T568B es el estandar mas utilizado en instalaciones comerciales en EE.UU.

## Patch cord, cable directo y cable cruzado

**Patch cord:** cable corta (tipicamente 0.5-5 metros) con los mismos extremos (ambos T568A o ambos T568B). Se utiliza para conectar dispositivos a patches de pared o switches.

**Cable directo (*straight-through*):** ambos extremos con el mismo estandar. Se utiliza para conectar dispositivos de tipos diferentes (PC a switch, switch a router).

**Cable cruzado (*crossover*):** un extremo T568A y el otro T568B. Se utiliza para conectar dispositivos del mismo tipo (PC a PC, switch a switch). Hoy en dia, la mayoria de dispositivos soportan Auto-MDIX y detectan automaticamente el tipo de cable necesario.

> La longitud maxima recomendada para par trenzado es de 100 metros en Ethernet. Esta limitacion se debe a la atenuacion de la senal y al tiempo de propagacion. Para distancias mayores, se utilizan repetidores (switches) o fibra optica.

