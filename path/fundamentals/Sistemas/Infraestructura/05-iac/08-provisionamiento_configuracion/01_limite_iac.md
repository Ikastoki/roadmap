# Limite de IaC: Provisionamiento vs Configuracion

La Infraestructura como Codigo excelencia en la creacion de recursos primitivos: una maquina virtual, un balanceador de carga, un bucket de almacenamiento o una subred. Estas son las piezas que componen el esqueleto de una plataforma. Sin embargo, una vez que ese recurso existe, el problema cambia de naturaleza. Configurar el interior de un sistema operativo — instalar paquetes, crear usuarios, ajustar parametros del kernel, desplegar aplicaciones — requiere un conjunto de herramientas y patrones fundamentalmente distintos a los que resuelven el aprovisionamiento.

La razon de esta distincion radica en el modelo de trabajo de las herramientas de IaC como Terraform o CloudFormation. Estas herramientas operan sobre una API externa: envian declaraciones de estado deseado a un proveedor cloud que interpreta y ejecuta los cambios. No tienen acceso directo al interior de la instancia. Cuando Terraform crea una `aws_instance`, el recurso queda registrado en el state file y la instancia esta levantada, pero Terraform no puede ejecutar comandos dentro de ella salvo que recurra a mecanismos auxiliares como provisioners, cuyo uso es considerado una practica degradada en la mayoria de los escenarios.

El limite entonces se traza en la frontera entre lo que el proveedor cloud puede hacer a traves de su API y lo que requiere acceso al sistema operativo. Crear un Security Group con reglas especificas es provisionamiento. Instalar Nginx dentro de la instancia es configuracion. Asignar un rol IAM a un recurso es provisionamiento. Configurar la política de auditoria de un servicio dentro del SO es configuracion.

```
+-------------------------------------------------------+
|                   PROVEEDOR CLOUD                     |
|                                                       |
|  +-----------------+     +-----------------------+    |
|  |   PROVISIONAMI- |     |   CONFIGURACION DEL   |    |
|  |   ENTI          |     |   SISTEMA OPERATIVO   |    |
|  |                 |     |                       |    |
|  | - VMs           |     | - Paquetes instalados |    |
|  | - Redes         |     | - Usuarios del SO     |    |
|  | - Balancers     |     | - Servicios internos  |    |
|  | - Buckets       |     | - Parametros kernel   |    |
|  | - IAM/Roles     |     | - Aplicaciones deploy  |    |
|  | - DNS           |     | - Certificados locales |    |
|  +-----------------+     +-----------------------+    |
|        API externa            Acceso SSH/rempoto      |
+-------------------------------------------------------+
```

Esta frontera tiene implicaciones practicas directas. Una herramienta de IaC puede garantizar que exista una instancia `t3.medium` con una IP privada en una subred determinada, pero no puede garantizar que Nginx este ejecutandose y respondiendo en el puerto 8080. Para ese segundo nivel de confiabilidad se necesitan mecanismos complementarios: bootstrapping con scripts, herramientas de gestion de configuracion, o imagenes pre-configuradas que eliminan la necesidad de configurar el SO en tiempo de ejecucion.

> En la practica, la mayoria de los proyectos combina ambos enfoques. IaC resuelve la capa de infraestructura (lo que se levanta) y herramientas complementarias resuelven la capa de configuracion (que hay dentro). La clave esta en entender donde termina una y empieza la otra para evitar fragilidad en los pipelines de despliegue.

La tendencia moderna, como se explorara en secciones posteriores, busca reducir al maximo la configuracion en tiempo de ejecucion mediante imagenes inmutables. La idea es que si la imagen ya contiene todo lo necesario, IaC solo necesita desplegarla sin preocuparse por el estado interno. Esto acerca la configuracion al paradigma declarativo del provisionamiento, pero requiere un pipeline de build separado para generar esas imagenes.

