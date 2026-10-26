# Herramientas de Despliegue y Despliegue Continuo

Mientras las herramientas de CI se encargan de compilar, testear y generar artefactos, las herramientas de despliegue asumen la responsabilidad de llevar esos artefactos a los entornos de produccion de forma segura y controlada. Este traspaso de responsabilidad marca la frontera entre la integracion continua y el despliegue continuo: la primera valida que el codigo funciona, la segunda garantiza que llega al usuario final. Las herramientas modernas de despliegue operan bajo el paradigma GitOps, donde Git se convierte en la unica fuente de verdad no solo del codigo, sino del estado deseado de toda la infraestructura.

## El paradigma GitOps

GitOps es un modelo operativo donde el estado completo del sistema (aplicaciones, configuraciones, infraestructura) se describe en repositorios Git, y herramientas especializadas se encargan de reconciliar continuamente el estado real con el estado deseado. Esta aproximacion resuelve uno de los problemas mas persistentes en operaciones: la drift, la divergencia gradual entre lo que se desplego manualmente y lo que documenta la configuracion.

En un modelo tradicional de despliegue, un operador ejecuta comandos para actualizar un sistema. En GitOps, el operador realiza un `git commit` con el cambio deseado, y un agente automatizado detecta el commit y aplica los cambios al sistema. Si el sistema se desvia del estado definido en Git (por ejemplo, por una intervencion manual o un fallo), el agente de reconciliacion restaura automaticamente el estado correcto.

Este modelo ofrece beneficios concretos: auditoria completa via historial de Git, posibilidad de rollback rapido con `git revert`, revisabilidad mediante pull requests, y reproducibilidad del estado de cualquier entorno en cualquier momento. GitOps no es solo una herramienta, es una metodologia que requiere disciplina en la gestion de repositorios y una comprension clara de la separacion entre configuracion de aplicaciones e infraestructura.

## ArgoCD

ArgoCD es un controlador de despliegue continuo diseñado especificamente para Kubernetes. Su funcion principal es monitorear repositorios Git y sincronizar automaticamente el estado de los recursos Kubernetes definidos en manifiestos Helm, Kustomize o plain YAML con el estado real del cluster.

La arquitectura de ArgoCD se compone de un servidor API, un servidor de interfaz web y un controlador de aplicaciones. El controlador realiza la tarea central: compara periodicamente el estado deseado (definido en Git) con el estado actual del cluster. Cuando detecta una diferencia, puede aplicar los cambios automaticamente (modo automatico) o esperar una aprobacion manual (modo manual).

ArgoCD organiza su modelo en `Application` y `ApplicationSet`. Una `Application` representa un despliegue individual que apunta a un repositorio, una revision y un destino (cluster y namespace). Un `ApplicationSet` genera multiples `Application` a partir de un template, util cuando se necesita desplegar la misma aplicacion en multiples clusters o namespaces con diferentes parametros.

Una de las capacidades mas valoradas de ArgoCD es la visualizacion del estado del despliegue: la interfaz web muestra el arbol de recursos Kubernetes, su estado de sincronizacion y el historial de revisiones desplegadas. Esto proporciona visibilidad inmediata sobre que版本 esta ejecutandose en cada entorno y permite realizar comparaciones entre revisiones.

> ArgoCD no reemplaza a las herramientas de CI. Se integra con ellas: la CI genera los artefactos y actualiza los manifiestos en el repositorio de configuracion, y ArgoCD detecta el cambio y despliega. Esta separacion de responsabilidades es un principio fundamental de GitOps.

## Flux

Flux es otra herramienta de despliegue continuo basada en GitOps para Kubernetes, desarrollada originalmente por Weaveworks y actualmente parte del CNCF (Cloud Native Computing Foundation). A diferencia de ArgoCD, que ofrece una interfaz web robusta, Flux sigue una arquitectura mas orientada a la linea de comandos y al ecosistema de Kubernetes nativo.

Flux esta compuesto por varios controladores independientes que trabajan juntos: `source-controller` gestiona la obtencion de manifiestos desde Git u otras fuentes; `kustomize-controller` aplica los manifiestos Kustomize al cluster; `helm-controller` gestiona los releases de Helm; y `notification-controller` maneja los webhooks entrantes y salientes para integracion con sistemas externos.

La Filosofia de Flux prioriza la composabilidad: cada controlador puede usarse de forma independiente o combinarse segun las necesidades. Esto permite construir pipelines de despliegue personalizados sin asumir una opinion sobre la estructura del proyecto. Flux tambien integra de forma nativa con herramientas de prueba como `conftest` y `kyverno`, permitiendo validar politicas de seguridad antes de que los cambios se apliquen al cluster.

El modelo de reconciliacion de Flux es pull-based, como ArgoCD: el agente dentro del cluster busca cambios en el repositorio, en lugar de recibir notificaciones push desde fuera. Este modelo es mas seguro porque el cluster nunca expone un endpoint de recepcion de cambios.

> Flux es particularmente adecuado para equipos que prefieren una herramienta ligera, composable y que se integre profundamente con el ecosistema Kubernetes, en lugar de una plataforma con interfaz grafica completa.

## Spinnaker

Spinnaker es una plataforma de despliegue continuo multiproveedor desarrollada originalmente por Netflix y actualmente mantenido como proyecto open source bajo el CNCF. A diferencia de ArgoCD y Flux, que se centran exclusivamente en Kubernetes, Spinnaker soporta despliegues a multiples objetivos: Kubernetes, AWS EC2, Google Compute Engine, Azure VMs, y otros.

La arquitectura de Spinnaker se organiza en pipelines de despliegue que constan de multiples stages (etapas). Cada stage representa una operacion concreta: ejecutar tests, construir una imagen, desplegar a un entorno, ejecutar pruebas de canary, o aprobar manualmente el despliegue. Las pipelines se definen a traves de la interfaz web o como codigo mediante el plugin Fiat, lo que permite versionarlas junto con el codigo.

Spinnaker destaca en escenarios de despliegue avanzado donde se requieren estrategias como canary deployments, blue/green deployments o despliegues con analisis automatico basado en metricas. Puede integrarse con sistemas de monitorizacion como Prometheus, Datadog o Stackdriver para evaluar automaticamente si un despliegue canary esta funcionando correctamente antes de completar la transicion.

El modelo de Spinnaker separa claramente la gestion de la infraestructura (que aplica los cambios) de la orquestacion de los flujos de trabajo (que decide cuando y como aplicarlos). Este separacion permite que multiples equipos compartan la misma plataforma de despliegue con diferentes politicas y niveles de permisos.

> Spinnaker tiene una curva de aprendizaje significativa y requiere una inversion considerable en su despliegue y mantenimiento. Es mas adecuado para organizaciones con necesidades de despliegue multiproveedor o estrategias de releaso avanzadas.

## Ansible y Terraform: Configuracion y Provisioning

Mientras ArgoCD, Flux y Spinnaker se centran en el despliegue de aplicaciones, Ansible y Terraform abordan las capas inferiores de la pila: la provision de infraestructura y la configuracion de servidores.

**Ansible** es una herramienta de gestion de configuracion que ejecuta playbooks YAML en nodos remotos via SSH. No requiere agentes instalados en los nodos objetivo, lo que reduce la friccion de adopcion. En el contexto de CI/CD, Ansible se usa frecuentemente para preparar entornos de ejecucion, instalar dependencias del sistema, configurar servicios o ejecutar migraciones de base de datos. Su modelo idempotente garantiza que ejecutar el mismo playbook multiple veces produce el mismo resultado sin efectos secundarios.

**Terraform** es una herramienta de provision de infraestructura que gestiona recursos en multiples proveedores cloud mediante archivos de configuracion declarativa. Terraform no ejecuta acciones sobre servidores existentes, sino que calcula el delta entre el estado actual y el estado deseado, y genera un plan de cambios que puede revisarse antes de aplicarse. En un pipeline CI/CD, Terraform se usa para crear o modificar la infraestructura donde se desplegaran las aplicaciones: redes, load balancers, clusters Kubernetes, bases de datos, etc.

La combinacion de estas herramientas con plataformas de despliegue GitOps crea una cadena completa: Terraform provisiona la infraestructura base, Ansible configura los sistemas operativos y servicios, y ArgoCD o Flux despliegan las aplicaciones. Cada capa puede gestionarse desde repositorios Git separados, manteniendo un registro completo de todos los cambios en la plataforma.

```yaml
# Ejemplo conceptual de un pipeline que integra todas las capas
etapas:
  - terraform apply    # Provision de infraestructura
  - ansible playbook   # Configuracion de servidores
  - argocd sync        # Despliegue de aplicaciones
```

> Ansible y Terraform no son directamente competidores: Ansible configura lo que ya existe, Terraform crea lo que no existe. En la practica, se complementan en lugar de sustituirse.

