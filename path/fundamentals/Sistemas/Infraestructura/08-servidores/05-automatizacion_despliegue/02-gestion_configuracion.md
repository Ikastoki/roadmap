# Gestion de Configuracion

Cuando un administrador configura un servidor manualmente, ejecuta una serie de comandos en un orden especifico: instala paquetes, edita archivos de configuracion, crea usuarios, inicia servicios. Si necesita repetir ese proceso en otro servidor, tiene dos opciones: volver a ejecutar los mismos comandos (con el riesgo de olvidar un paso o cometer un error de tipeo) o automatizarlo con un script. Los scripts shell, como se vio en la seccion anterior, funcionan bien para tareas simples pero escalan mal: cada servidor requiere su propio script, las dependencias entre tareas se gestionan manualmente, y no hay forma declarativa de describir el estado final deseado. La gestion de configuracion resuelve estos problemas introduciendo un modelo diferente: en lugar de describir **como** configurar un sistema, se describe **como deberia verse** el sistema configurado. La herramienta se encarga de calcular los pasos necesarios y ejecutarlos de forma segura.

## El modelo declarativo frente al imperativo

Un script bash es imperativo: "instala nginx, copia este archivo, reinicia el servicio". Cada paso es una instruccion que el sistema ejecuta en orden. Si el paso 3 falla, los pasos 4 y 5 no se ejecutan, pero el sistema queda en un estado parcialmente configurado. Un enfoque declarativo dice: "el servidor debe tener nginx instalado, este archivo de configuracion, y el servicio activo". La herramienta compara el estado actual con el estado deseado y aplica solo los cambios necesarios. Si nginx ya esta instalado, no lo reinstala. Si el archivo de configuracion ya es correcto, no lo modifica. Esto es idempotencia por diseno, no por construccion manual.

## Arquitectura agent vs agentless

Las herramientas de gestion de configuracion se dividen en dos categorias segun como se comunican con los servidores gestionados.

### Arquitectura con agente (Puppet, Chef, SaltStack en modo agent)

En este modelo, cada servidor gestionado ejecuta un **agente**: un proceso daemon que se ejecuta periodicamente (generalmente cada 30 minutos) o recibe push desde un servidor central. El agente consulta el servidor de configuracion, descarga las reglas aplicables a ese nodo, calcula los cambios necesarios y los aplica localmente.

**Ventajas:**
- El agente puede ejecutar tareas complejas con dependencias locales.
- Si la conexion de red se interrumpe, el agente sigue ejecutando las reglas descargadas.
- Permite ejecucion continua sin intervencion externa.

**Desventajas:**
- Requiere instalar y mantener un agente en cada servidor.
- Cada servidor consume recursos (CPU, memoria) para el agente.
- La version del agente debe ser compatible con el servidor central, lo que agrega complejidad de mantenimiento.
- Los servidores temporales (containers, instancias efimeras) necesitan instalar el agente antes de ser gestionados.

### Arquitectura sin agente (Ansible)

Ansible no instala nada en los servidores gestionados. Se comunica a traves de SSH, ejecutando modulos Python en el servidor remoto que se eliminan automaticamente despues de cada ejecucion. El servidor de control mantiene el inventario y los playbooks, y ejecuta las tareas conectandose por SSH a cada nodo.

**Ventajas:**
- No requiere instalacion de agentes.
- Funciona con cualquier servidor que tenga SSH y Python.
- No hay proceso daemon consumiendo recursos.
- Los servidores temporales se gestionan inmediatamente.

**Desventajas:**
- La ejecucion es secuencial o limitadamente paralela (controlado por forks).
- Requiere conectividad SSH activa durante toda la ejecucion.
- No puede ejecutar tareas si la red se cae.

### SaltStack: modelo hibrido

SaltStack es notable porque soporta ambos modos. En modo minion (agente), un daemon en cada servidor recibe instrucciones del servidor Salt Master. En modo SSH, funciona sin agente, similar a Ansible. Esto permite usar la misma herramienta en distintos escenarios, aunque agrega complejidad de configuracion.

## Herramientas principales

### Puppet

Puppet utiliza un lenguaje propio (DSL) para describir el estado deseado del sistema. Las configuraciones se organizan en **manifests** (archivos `.pp`) que declaran **recursos** como paquetes, archivos y servicios. Un **node** (servidor) se evalua contra los manifests correspondientes y Puppet genera un catalogo de cambios antes de aplicarlos.

El ciclo de ejecucion de Puppet es: el agente envia informacion del sistema al servidor Puppet, el servidor compila un catalogo basado en los manifests, el agente descarga el catalogo y lo aplica. Este ciclo repetitivo garantiza que el sistema converja hacia el estado deseado, incluso si algo lo desvia manualmente.

```puppet
package { 'nginx':
  ensure => installed,
}

file { '/etc/nginx/nginx.conf':
  ensure  => file,
  content => template('modulo/nginx.conf.erb'),
  notify  => Service['nginx'],
}

service { 'nginx':
  ensure => running,
  enable => true,
}
```

Los bloques anteriores declaran tres recursos con sus propiedades. La directiva `notify` establece una dependencia: si el archivo de configuracion cambia, el servicio se reinicia. Puppet resuelve la cadena de dependencias automaticamente.

### Chef

Chef utiliza Ruby como lenguaje de escritura. Las configuraciones se organizan en **cookbooks** que contienen **recipes** (recetas). Cada receta describe un conjunto de recursos y como deben configurarse. Chef sigue un modelo de infraestructura como codigo donde los cookbooks se versionan, prueban y distribuyen como cualquier proyecto de software.

La diferencia principal con Puppet es el lenguaje: Ruby permite logica de programacion compleja (condicionales, bucles, clases) directamente en las recetas, lo cual es mas flexible pero tambien mas propenso a errores si no se aplican praticas de testing.

### Ansible

Ansible merece atencion particular porque combina la potencia de un modelo declarativo con la simplicidad de no requerir agentes. Sus configuraciones se escriben en YAML, lo cual es mas legible que los DSL de Puppet o Chef. Un **playbook** define hosts (servidores) y tareas que deben ejecutarse en ellos. Las tareas invocan **modulos** que realizan acciones especificas: instalar paquetes, gestionar usuarios, copiar archivos, reiniciar servicios.

La ausencia de agente elimina una capa completa de complejidad operativa. No hay daemon que actualizar, no hay configuracion de certificados de agente, no hay dependencia de un proceso en cada nodo. El acceso SSH con llave publica es el unico requisito previo, y es algo que cualquier servidor de produccion ya tiene configurado.

## Convergencia y estado deseado

El concepto fundamental que une a todas estas herramientas es la **convergencia**. Un sistema configurado con herramientas de gestion de configuracion esta constantemente convergiendo hacia el estado declarado. Si alguien modifica manualmente un archivo de configuracion, la herramienta lo restaura en la proxima ejecucion. Si se instala un paquete no autorizado, la herramienta lo desinstala. Este comportamiento es esencial en entornos donde multiples personas tienen acceso a los servidores: la herramienta actua como autoridad unica del estado correcto.

