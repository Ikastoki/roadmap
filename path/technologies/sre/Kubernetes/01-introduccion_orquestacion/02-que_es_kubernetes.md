# Que es Kubernetes

Kubernetes es una plataforma open source para la orquestacion de contenedores que automatiza el despliegue, el escalado y la operacion de aplicaciones en un cluster de servidores. No es un simple gestor de contenedores como Docker Compose, ni una herramienta de aprovisionamiento como Ansible o Terraform. Kubernetes define un modelo declarativo de operaciones: el administrador especifica el estado deseado del sistema y la plataforma ejecuta las acciones necesarias para alcanzarlo, manteniendolo en el tiempo.

Es, en esencia, un sistema operativo para el centro de datos. Del mismo modo que un sistema operativo abstrae el hardware de una maquina individual y gestiona sus recursos (CPU, memoria, disco), Kubernetes abstrae un conjunto de servidores y gestiona los recursos del cluster como un unico plano de computo. Las aplicaciones no se despliegan en maquinas concretas, sino en el cluster, y Kubernetes decide donde y como ejecutarlas.

## El origen: de Borg a Kubernetes

Kubernetes no nacio en un laboratorio academico ni como un proyecto comunitario desde cero. Es el fruto de mas de una decada de experiencia operativa en Google. La compania desarrollo internamente un sistema llamado Borg a principios de los 2000 para gestionar sus cargas de trabajo a escala planetaria: miles de aplicaciones ejecutandose en cientos de miles de maquinas, con requisitos de fiabilidad y eficiencia extremos.

Borg resolvia problemas que el ecosistema de contenedores apenas empezaba a enfrentar en 2014: empaquetado optimizado de procesos en maquinas compartidas, priorizacion de cargas de trabajo, tolerancia a fallos a nivel de cluster, y definicion declarativa del estado deseado. Cuando Google decidio compartir esta experiencia con la comunidad, no libero Borg directamente. En su lugar, reescribio el sistema desde cero basandose en las lecciones aprendidas, y lo llamo Kubernetes (del griego, "timonel" o "capitan de barco").

Kubernetes fue anunciado como proyecto open source en junio de 2014. El diseno inicial fue liderado por Craig McLuckie, Joe Beda y Brendan Burns, los mismos ingenieros que habian trabajado en Borg. Desde el principio, el proyecto adopto un enfoque modular y extensible que contrastaba con la arquitectura monolitica de Borg.

## La donacion a la CNCF

En 2015, Google dono Kubernetes a la recien creada Cloud Native Computing Foundation (CNCF), una entidad de la Linux Foundation dedicada a promover tecnologias nativas de la nube. Esta donacion fue un movimiento estrategico que cambio el panorama del software de infraestructura.

Al alojarse en la CNCF, Kubernetes se desvinculo del control de una unica empresa. Cualquier organizacion podia contribuir al proyecto sin temor a decisiones unilaterales de Google. Amazon, Microsoft, IBM, Red Hat, VMware y cientos de empresas mas se unieron al desarrollo. Este gobierno neutral y transparente fue clave para que Kubernetes se convirtiera en el estandar de facto de la orquestacion de contenedores.

La CNCF no solo alberga Kubernetes. Gestiona un ecosistema completo de proyectos que cubren todas las capas de una aplicacion nativa de la nube: redes (CNI), almacenamiento (CSI), service mesh (Envoy, Istio), observabilidad (Prometheus, Jaeger), registros de contenedores (Harbor), y muchos mas. Kubernetes actua como el nucleo sobre el que se construye este ecosistema.

```bash
# La CNCF clasifica los proyectos por nivel de madurez
# Graduated: Kubernetes, Prometheus, Envoy, CoreDNS, containerd, Fluentd, Jaeger, etc.
# Incubating: etcd, gRPC, Linkerd, Helm, Harbor, etc.
# Sandbox: proyectos en fase inicial
```

> Que Kubernetes este en la CNCF no es un detalle burocratico. Implica que el proyecto cumple con estandares estrictos de gobernanza, transparencia y seguridad. Cualquier empresa puede adoptarlo sabiendo que no dependera de un proveedor concreto.

## El estandar de facto

Ocho anos despues de su lanzamiento, Kubernetes es el estandar indiscutible de orquestacion de contenedores. Todos los proveedores de nube ofrecen Kubernetes como servicio gestionado: Google Kubernetes Engine (GKE), Amazon Elastic Kubernetes Service (EKS), Azure Kubernetes Service (AKS), DigitalOcean Kubernetes, IBM Cloud Kubernetes Service, y decenas mas. Incluso servicios propietarios como OpenShift (Red Hat) o Rancher (SUSE) se construyen sobre Kubernetes.

Convertirse en estandar no significa ausencia de alternativas. Docker Swarm y Apache Mesos existian antes de Kubernetes y ofrecian soluciones validas. Docker Swarm destacaba por su simplicidad: integrado nativamente con la CLI de Docker, permitia crear un cluster con pocos comandos. Apache Mesos ofrecia una gestion de recursos mas granular y soportaba cargas de trabajo no containerizadas. Pero ninguna logro la combinacion de flexibilidad, portabilidad y ecosistema que ofrece Kubernetes.

La portabilidad es quizas su ventaja mas solida. Un manifiesto de Kubernetes escrito para un cluster on-premise funciona sin cambios en GKE, EKS o AKS. Una aplicacion disenada siguiendo las practicas nativas de la nube puede migrar de un proveedor a otro con modificaciones minimas. Esta portabilidad reduce el vendor lock-in y da a las organizaciones poder de negociacion con sus proveedores de infraestructura.

## La arquitectura en esencia

Kubernetes sigue una arquitectura de plano de control y nodos de trabajo. El plano de control (control plane) alberga los componentes que toman decisiones globales sobre el cluster: el API server, la base de datos de estado (etcd), el scheduler y los controladores. Los nodos de trabajo ejecutan los contenedores y reportan su estado al plano de control.

Esta separacion no es arbitraria. El plano de control es el cerebro del cluster; si falla, el cluster deja de responder a nuevos cambios, aunque los contenedores en ejecucion continuan funcionando. Por eso en entornos productivos el plano de control se despliega de forma redundante en varios servidores, y etcd requiere al menos tres nodos para garantizar consenso.

```bash
# Componentes del control plane y el nodo worker
# Control plane:
#   kube-apiserver     -> Punto unico de interaccion (REST)
#   etcd               -> Almacenamiento clave-valor distribuido
#   kube-scheduler     -> Asigna pods a nodos
#   kube-controller-manager -> Ejecuta controladores
#   cloud-controller-manager -> Integracion con proveedor cloud
#
# Nodo worker:
#   kubelet            -> Agente del nodo
#   kube-proxy         -> Reglas de red
#   container runtime  -> Docker, containerd, CRI-O
```

Kubernetes no ejecuta contenedores directamente. Delega esa responsabilidad a un runtime de contenedores que implemente la CRI (Container Runtime Interface). Los runtimes mas comunes son containerd (el mas utilizado, adoptado por Docker y Kubernetes por defecto) y CRI-O (disenado especificamente para Kubernetes). Esta abstraccion permite que Kubernetes funcione con cualquier runtime compatible, incluyendo alternativas como gVisor o Kata Containers que proporcionan capas adicionales de aislamiento.

