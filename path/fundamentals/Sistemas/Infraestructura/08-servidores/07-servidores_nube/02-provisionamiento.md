# Provisionamiento de instancias

Provisionar una instancia en la nube es el acto de crear y poner en marcha un servidor virtual listo para recibir carga de trabajo. A diferencia del provisionamiento fisico — que implica encadenar tareas manuales con dependencias criticas — el provisionamiento en la nube es una operacion atomica que completa todo el ciclo (hardware, SO, red, almacenamiento) en cuestion de minutos. Sin embargo, la velocidad del proceso no elimina la necesidad de planificacion: cada decision tomada durante el provisionamiento — imagen base, tipo de instancia, zona, configuracion de red — tiene consecuencias directas en coste, rendimiento y seguridad.

## Creacion desde la consola web

Los proveedores de nube ofrecen consolas web graficas que permiten crear instancias mediante un asistente paso a paso. El flujo tipico incluye seleccionar la imagen base del sistema operativo, elegir el tipo y tamano de instancia, configurar la clave de acceso SSH, definir las reglas de seguridad, asignar almacenamiento y revisar un resumen antes de lanzar. La consola es ideal para prototipado, pruebas o administradores que estan aprendiendo el entorno, pero presenta limitaciones claras: no es reproducible, no permite control de versiones y requiere interaccion manual en cada paso.

> Las consolas web son herramientas de aprendizaje excelentes pero peligrosas en produccion. Una configuracion aplicada manualmente no deja registro automatico de quien hizo que cambio ni por que razon, lo que complica auditorias y reproduccion de entornos.

## Creacion desde la CLI y APIs

La verdadera potencia del provisionamiento en la nube reside en las interfaces de linea de comandos (AWS CLI, Azure CLI, gcloud) y las APIs REST subyacentes. Cada accion que se puede realizar desde la consola web tiene un equivalente en la CLI, pero con ventajas adicionales: las comandos pueden automatizarse en scripts, integrarse en pipelines de CI/CD, versionarse en repositorios y ejecutarse de forma idempotente. Un script de provisionamiento ejecutado mil veces produce el mismo resultado, mientras que la accion manual puede variar segun quien la ejecute.

```
aws ec2 run-instances \
  --image-id ami-0abcdef1234567890 \
  --instance-type t3.micro \
  --key-name mi-clave \
  --security-group-ids sg-0123456789abcdef0 \
  --subnet-id subnet-0123456789abcdef0
```

Este comando crea una instancia EC2 en AWS con los parametros especificados. La idempotencia del comando permite re-ejecutarlo sin crear instancias duplicadas si se proporciona un token de idempotencia, una caracteristica fundamental para sistemas de provisionamiento robustos.

## Imagenes base y consistencia

Una imagen base (AMI en AWS, Azure Image, GCP Image) es una plantilla que contiene el sistema operativo preconfigurado con software, configuraciones y datos iniciales. Utilizar una imagen base en lugar de instalar el sistema operativo desde cero cada vez garantiza consistencia: todas las instancias derivadas de la misma imagen comenzaran con el mismo estado base, eliminando variaciones no controladas que generan bugs dificiles de diagnosticar.

El proceso tipico de creacion de imagenes base implica levantar una instancia, instalar y configurar todo el software necesario (servidores web, bases de datos, agentes de monitoreo), personalizar los archivos de configuracion, y finalmente crear una imagen a partir de esa instancia. Esta imagen puede distribuirse entre regiones o cuentas,.Versionarse con etiquetas descriptivas y compartirse con otros equipos de la organizacion.

> Las imagenes base funcionan como el concepto de "golden image" en DevOps: un template conocido, testado y versionado que sirve como punto de partida fiable. La alternativa — configurar cada servidor manualmente o con scripts ad-hoc — conduce a la "configuracion deriva", donde entornos supuestamente identicos se comportan de forma diferente.

La consistencia que proporcionan las imagenes base es especialmente critica en entornos de produccion donde una diferencia minima en la version de una libreria o en un parametro de configuracion puede provocar un fallo en cascada. Las imagenes tambien facilitan el cumplimiento normativo, ya que se puede certificar que una imagen contiene unicamente software aprobado por la organizacion.

