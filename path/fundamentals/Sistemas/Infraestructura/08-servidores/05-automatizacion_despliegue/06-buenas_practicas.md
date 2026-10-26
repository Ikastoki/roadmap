# Buenas Practicas en Automatizacion de Despliegue

Configurar un servidor "a mano" parece rapido la primera vez. Un administrador SSH, ejecuta algunos comandos, instala lo que necesita, ajusta configuraciones, y el servidor funciona. Pero ese conocimiento vive exclusivamente en la cabeza del administrador. Cuando el servidor falla y debe recrearse, cuando se necesita un segundo servidor identico, o cuando el administrador no esta disponible, la organisation descubre que la configuracion manual es una forma de deuda tecnica: un atajo que genera intereses compuestos en forma de tiempo perdido, errores de reproduccion y dependencia de personas especificas.

## Por que la configuracion manual es deuda tecnica

Un servidor configurado manualmente es un **snowflake**: unico, irreproducible y frágil. Si funciona, nadie sabe exactamente por que funciona. Si falla, nadie sabe exactamente que cambio. Los problemas concretos son multiples:

- **Sin historial**: no hay registro de que comandos se ejecutaron, en que orden, ni por que motivo. Si el servidor se configuro hace dos anos, nadie puede replicar exactamente los mismos pasos.
- **Sin reproducibilidad**: configurar un segundo servidor identico requiere que el administrador recuerde todos los pasos. Los archivos de configuracion que edito a mano, los paquetes que instalo, las variables de entorno que definio, los permisos que asigno. Cualquier omision genera diferencias silenciosas entre servidores que deberian ser identicos.
- **Sin reversibilidad**: si un cambio de configuracion causa un problema, no hay forma de saber exactamente que se cambio ni de volver al estado anterior. El administrador debe deducir el problema y corregirlo manualmente.
- **Sin escalabilidad**: configurar 10 servidores manualmente toma 10 veces mas que configurar uno. No hay economia de escala porque cada configuracion es un evento unico.

## Todo en codigo, versionado en Git

La solucion es representar toda la configuracion como archivos de texto que se almacenan en un repositorio Git. Esto incluye:

- **Scripts de provision**: modulos Terraform que crean servidores, redes y recursos cloud.
- **Playbooks de configuracion**: archivos Ansible que instalan y configuran el software.
- **Archivos de configuracion**: templates Jinja2 que generan configuraciones personalizadas.
- **Scripts de despliegue**: pipelines CI/CD que orquestan todo el proceso.

Cada cambio en estos archivos se registra en Git con un mensaje descriptivo, un autor y una marca temporal. Si algo falla despues de un cambio, `git bisect` permite identificar exactamente que commit introdujo el problema. `git revert` permite deshacer el cambio de forma limpia. La revision de codigo garantiza que al menos otra persona reviso los cambios antes de que se apliquen en produccion.

> La practica de revisar cambios de infraestructura antes de aplicarlos es tan critica como la revision de codigo de la aplicacion. Un error en un playbook de Ansible puede afectar a todos los servidores simultaneamente, con un impacto potencialmente mayor que un bug en la aplicacion.

## Principios clave

### Parametrizacion sobre hardcoding

Las configuraciones deben parametrizarse para funcionar en multiples entornos (desarrollo, staging, produccion). Un playbook que contiene la IP del servidor de base de datos hardcodeado solo funciona para un entorno. Un playbook que referencia una variable `db_host` funciona para cualquiera.

### Testing de infraestructura

Los playbooks y modulos de Terraform deben probarse antes de aplicarse en produccion. Herramientas como `molecule` permiten probar playbooks de Ansible en containers, verificando que los cambios se aplican correctamente sin afectar servidores reales. Los modulos de Terraform se prueban con `terraform plan` antes de `terraform apply`.

### Despliegue automatizado y automatizable

Ningun paso del despliegue debe requerir intervencion manual. Si un paso necesita aprobacion humana, que sea un gate en el pipeline con boton de aprobacion, no un correo electronico que alguien lea y ejecute manualmente. La consistencia del proceso es mas importante que la velocidad de un despliegue individual.

### Observabilidad del proceso

Cada ejecucion del pipeline debe generar logs que permitan reconstruir que paso, cuando paso, y que resultado tuvo. Si un despliegue falla a las 3:00 AM, el ingeniero on-call necesita ver el log completo para diagnosticar el problema sin despertar al resto del equipo.

### Prueba de despliegue en entorno de staging

Nunca se despliega directamente a produccion sin pasar primero por un entorno de staging que sea una replica fiel. El staging detecta problemas de configuracion, dependencias faltantes y conflictos que no se manifiestan en el entorno de desarrollo.

