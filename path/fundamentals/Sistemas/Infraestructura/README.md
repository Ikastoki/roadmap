# Infraestructura

Esta sección reúne conocimientos y prácticas para versionar código, virtualizar sistemas, ejecutar y orquestar contenedores, automatizar infraestructura y operar servicios y servidores.

> [!WARNING]
> El material está en desarrollo y puede contener errores, lagunas o ejemplos que requieran adaptación al entorno. Contrasta los procedimientos con la documentación oficial antes de aplicarlos, especialmente en sistemas en producción.

## Índice

- [Control de versiones](#control-de-versiones)
- [Virtualización](#virtualización)
- [Contenedores](#contenedores)
- [Orquestadores](#orquestadores)
- [Infraestructura como código (IaC)](#infraestructura-como-código-iac)
- [Monitorización y observabilidad](#monitorización-y-observabilidad)
- [Integración y despliegue continuos (CI/CD)](#integración-y-despliegue-continuos-cicd)
- [Servidores](#servidores)

## Control de versiones

### Introduccion VCS

- [Qué es el control de versiones](01-vcs/01-introduccion_vcs/01-que_es_control_versiones.md)
- [Problemas que resuelve el control de versiones](01-vcs/01-introduccion_vcs/02-problemas_resuelve.md)
- [Casos de uso mas alla del codigo fuente](01-vcs/01-introduccion_vcs/03-casos_uso.md)
- [Evolucion historica de los VCS](01-vcs/01-introduccion_vcs/04-evolucion_historica.md)
- [Instalacion y configuracion inicial](01-vcs/01-introduccion_vcs/05-instalacion_configuracion.md)

### Fundamentos Distribuidos

- [Arquitectura conceptual de Git](01-vcs/02-fundamentos_distribuidos/01-arquitectura_conceptual.md)
- [Primeros pasos en un repositorio](01-vcs/02-fundamentos_distribuidos/02-primeros_pasos.md)
- [Ciclo basico de trabajo en Git](01-vcs/02-fundamentos_distribuidos/03-ciclo_basico.md)
- [Gestion de exclusiones](01-vcs/02-fundamentos_distribuidos/04-gestion_exclusiones.md)
- [Visualizacion y comparacion de cambios](01-vcs/02-fundamentos_distribuidos/05-visualizacion_comparacion.md)
- [Primeros errores comunes y su correccion](01-vcs/02-fundamentos_distribuidos/06-errores_comunes.md)

### Gestion Historial Ramas

- [Concepto de rama](01-vcs/03-gestion_historial_ramas/01-concepto_rama.md)
- [Trabajo con ramas](01-vcs/03-gestion_historial_ramas/02-trabajo_con_ramas.md)
- [Estrategias de ramificacion](01-vcs/03-gestion_historial_ramas/03-estrategias_ramificacion.md)
- [Reescritura de historial](01-vcs/03-gestion_historial_ramas/04-reescritura_historial.md)
- [Etiquetado (Tags)](01-vcs/03-gestion_historial_ramas/05-etiquetado.md)

### Colaboracion Remotos

- [Concepto de repositorio remoto](01-vcs/04-colaboracion_remotos/01-concepto_remoto.md)
- [Operaciones de sincronizacion](01-vcs/04-colaboracion_remotos/02-operaciones_sincronizacion.md)
- [Flujo colaborativo basico](01-vcs/04-colaboracion_remotos/03-flujo_colaborativo.md)
- [Manejo de ramas remotas](01-vcs/04-colaboracion_remotos/04-ramas_remotas.md)
- [Colisiones y buenas practicas de red](01-vcs/04-colaboracion_remotos/05-buenas_practicas_red.md)

### Plataformas Colaboracion

- [Valor añadido mas alla del VCS](01-vcs/05-plataformas_colaboracion/01-valor_anadido.md)
- [Flujo de trabajo con solicitudes de fusion](01-vcs/05-plataformas_colaboracion/02-flujo_solicitudes.md)
- [Forks y contribucion a proyectos de terceros](01-vcs/05-plataformas_colaboracion/03-forks_contribucion.md)
- [Documentacion y estandares del repositorio](01-vcs/05-plataformas_colaboracion/04-documentacion_estandares.md)
- [Markdown](01-vcs/05-plataformas_colaboracion/05-markdown.md)
- [Herramientas de linea de comandos de la plataforma](01-vcs/05-plataformas_colaboracion/06-herramientas_cli.md)
- [Privacidad y licencias](01-vcs/05-plataformas_colaboracion/07-privacidad_licencias.md)

### Herramientas Buenas Practicas

- [Mensajes de commit efectivos](01-vcs/06-herramientas_buenas_practicas/01-mensajes_commit.md)
- [CLI vs. GUI para Git](01-vcs/06-herramientas_buenas_practicas/02-cli_vs_gui.md)
- [Personalizacion del entorno de Git](01-vcs/06-herramientas_buenas_practicas/03-personalizacion_entorno.md)
- [Ganchos (Hooks) del sistema](01-vcs/06-herramientas_buenas_practicas/04-ganchos_hooks.md)
- [Integracion con CI/CD](01-vcs/06-herramientas_buenas_practicas/05-integracion_cicd.md)
- [Auditoria, recuperacion y limpieza](01-vcs/06-herramientas_buenas_practicas/06-auditoria_recuperacion.md)
- [Autenticacion segura con SSH](01-vcs/06-herramientas_buenas_practicas/07-ssh_autenticacion.md)

## Virtualización

### Introduccion

- [Que es la virtualizacion](02-virtualizacion/01-introduccion/01-que_es_virtualizacion.md)
- [Historia y evolucion de la virtualizacion](02-virtualizacion/01-introduccion/02-historia_evolucion.md)
- [Beneficios clave de la virtualizacion](02-virtualizacion/01-introduccion/03-beneficios_clave.md)
- [Tipos de virtualizacion](02-virtualizacion/01-introduccion/04-tipos_virtualizacion.md)
- [Componentes esenciales de la virtualizacion](02-virtualizacion/01-introduccion/05-componentes_esenciales.md)
- [Escenarios de uso inicial](02-virtualizacion/01-introduccion/06-escenarios_uso.md)

### Hipervisores Tipo1

- [Que es un hipervisor de Tipo 1](02-virtualizacion/02-hipervisores_tipo1/01-que_es_hipervisor_tipo1.md)
- [Arquitectura interna de los hipervisores Tipo 1](02-virtualizacion/02-hipervisores_tipo1/02-arquitectura_interna.md)
- [Caracteristicas de los hipervisores nativos](02-virtualizacion/02-hipervisores_tipo1/03-caracteristicas.md)
- [Ventajas y desventajas de los hipervisores Tipo 1](02-virtualizacion/02-hipervisores_tipo1/04-ventajas_desventajas.md)
- [Instalacion y operacion de hipervisores Tipo 1](02-virtualizacion/02-hipervisores_tipo1/05-instalacion_operacion.md)

### Hipervisores Tipo2

- [Que es un hipervisor de Tipo 2](02-virtualizacion/03-hipervisores_tipo2/01-que_es_hipervisor_tipo2.md)
- [Arquitectura interna de los hipervisores Tipo 2](02-virtualizacion/03-hipervisores_tipo2/02-arquitectura_interna.md)
- [Caracteristicas de los hipervisores alojados](02-virtualizacion/03-hipervisores_tipo2/03-caracteristicas.md)
- [Funcionalidades comunes de los hipervisores Tipo 2](02-virtualizacion/03-hipervisores_tipo2/04-funcionalidades_comunes.md)
- [Ventajas y desventajas de los hipervisores Tipo 2](02-virtualizacion/03-hipervisores_tipo2/05-ventajas_desventajas.md)
- [Instalacion y uso tipico de hipervisores Tipo 2](02-virtualizacion/03-hipervisores_tipo2/06-instalacion_uso.md)

### Gestion VMs

- [Ciclo de vida de una VM](02-virtualizacion/04-gestion_vms/01-ciclo_vida_vm.md)
- [Creacion de una maquina virtual](02-virtualizacion/04-gestion_vms/02-creacion_vm.md)
- [Herramientas de integracion](02-virtualizacion/04-gestion_vms/03-herramientas_integracion.md)
- [Gestion de recursos](02-virtualizacion/04-gestion_vms/04-gestion_recursos.md)
- [Monitoreo y rendimiento](02-virtualizacion/04-gestion_vms/05-monitoreo_rendimiento.md)
- [Migracion y portabilidad](02-virtualizacion/04-gestion_vms/06-migracion_portabilidad.md)

### Virtualizacion Nube

- [La nube como evolucion de la virtualizacion](02-virtualizacion/05-virtualizacion_nube/01-nube_evolucion.md)
- [Caracteristicas de las VMs en la nube](02-virtualizacion/05-virtualizacion_nube/02-caracteristicas_vms_nube.md)
- [Automatizacion y escalado](02-virtualizacion/05-virtualizacion_nube/03-automatizacion_escalado.md)
- [Diferencias clave vs. virtualizacion local](02-virtualizacion/05-virtualizacion_nube/04-diferencias_vs_local.md)
- [Consideraciones de seguridad y costos](02-virtualizacion/05-virtualizacion_nube/05-seguridad_costos.md)

### Comparacion

- [Virtualizacion vs. contenedores](02-virtualizacion/06-comparacion/01-virtualizacion_vs_contenedores.md)
- [Virtualizacion vs. emulacion](02-virtualizacion/06-comparacion/02-virtualizacion_vs_emulacion.md)
- [Maquinas virtuales vs. entornos sandbox](02-virtualizacion/06-comparacion/03-vm_vs_sandbox.md)
- [Serverless y funciones (FaaS)](02-virtualizacion/06-comparacion/04-serverless_faaS.md)
- [Cuando usar que](02-virtualizacion/06-comparacion/05-cuando_usar_que.md)
- [Tendencias: micro-VMs y seguridad reforzada](02-virtualizacion/06-comparacion/06-tendencias_microvm.md)

### Casos Uso

- [Entornos de desarrollo y pruebas](02-virtualizacion/07-casos_uso/01-entornos_desarrollo.md)
- [Servidores locales (homelab)](02-virtualizacion/07-casos_uso/02-servidores_locales.md)
- [Migracion de sistemas legacy](02-virtualizacion/07-casos_uso/03-migracion_legacy.md)
- [Aislamiento de seguridad](02-virtualizacion/07-casos_uso/04-aislamiento_seguridad.md)
- [Empresas y centros de datos](02-virtualizacion/07-casos_uso/05-empresas_centros_datos.md)

## Contenedores

### Introduccion

- [Que es un contenedor](03-contenedores/01-introduccion/01-que_es_contenedor.md)
- [Problemas que resuelven los contenedores](03-contenedores/01-introduccion/02-problemas_resuelven.md)
- [Contenedores vs. maquinas virtuales](03-contenedores/01-introduccion/03-contenedores_vs_vms.md)
- [Historia y evolucion de la contenerizacion](03-contenedores/01-introduccion/04-historia_evolucion.md)
- [Beneficios clave del estandar de contenedores](03-contenedores/01-introduccion/05-beneficios_estandar.md)
- [Glosario de conceptos clave](03-contenedores/01-introduccion/06-glosario_conceptos.md)
- [Instalacion inicial](03-contenedores/01-introduccion/07-instalacion_inicial.md)

### Arquitectura Basica

- [Componentes del ecosistema de contenedores](03-contenedores/02-arquitectura_basica/01-componentes_ecosistema.md)
- [Tecnologias subyacentes en Linux](03-contenedores/02-arquitectura_basica/02-tecnologias_subyacentes.md)
- [Arquitectura cliente-servidor vs. daemonless](03-contenedores/02-arquitectura_basica/03_arquitectura_client_server_vs_daemonless.md)
- [Contenedores como procesos](03-contenedores/02-arquitectura_basica/04-contenedores_como_procesos.md)
- [Contenedores en diferentes sistemas operativos](03-contenedores/02-arquitectura_basica/05_contenedores_diferentes_so.md)
- [Seguridad basica de contenedores](03-contenedores/02-arquitectura_basica/06-seguridad_basica.md)

### Gestion Imagenes

- [Que es una imagen de contenedor](03-contenedores/03-gestion_imagenes/01_que_es_imagen.md)
- [Registros de imagenes (Registries)](03-contenedores/03-gestion_imagenes/02-registros_imagenes.md)
- [Operaciones basicas con imagenes](03-contenedores/03-gestion_imagenes/03-operaciones_basicas.md)
- [Etiquetas (tags) y versiones](03-contenedores/03-gestion_imagenes/04-etiquetas_versiones.md)
- [Inspeccion de imagenes](03-contenedores/03-gestion_imagenes/05-inspeccion_imagenes.md)
- [Buenas practicas de imagenes](03-contenedores/03-gestion_imagenes/06-buenas_practicas.md)

### Ejecucion Gestion

- [Ejecutar un contenedor](03-contenedores/04-ejecucion_gestion/01-ejecutar_contenedor.md)
- [Ciclo de vida de un contenedor](03-contenedores/04-ejecucion_gestion/02-ciclo_vida_contenedor.md)
- [Gestion basica de contenedores](03-contenedores/04-ejecucion_gestion/03-gestion_basica.md)
- [Acceso y diagnostico de contenedores](03-contenedores/04-ejecucion_gestion/04-acceso_diagnostico.md)
- [Recursos y limites](03-contenedores/04-ejecucion_gestion/05-recursos_limites.md)
- [Eliminacion automatica](03-contenedores/04-ejecucion_gestion/06-eliminacion_automatica.md)

### Construccion Imagenes

- [Containerfile / Dockerfile](03-contenedores/05-construccion_imagenes/01-dockerfile_containerfile.md)
- [Instrucciones clave del Dockerfile](03-contenedores/05-construccion_imagenes/02-instrucciones_clave.md)
- [Construccion de la imagen](03-contenedores/05-construccion_imagenes/03-construccion_imagen.md)
- [Optimizacion de la construccion](03-contenedores/05-construccion_imagenes/04-optimizacion_construccion.md)
- [Errores comunes de construccion](03-contenedores/05-construccion_imagenes/05-errores_comunes.md)

### Almacenamiento

- [El problema de la efimeridad](03-contenedores/06-almacenamiento/01-problema_efimeridad.md)
- [Tipos de almacenamiento](03-contenedores/06-almacenamiento/02-tipos_almacenamiento.md)
- [Gestion de volumenes](03-contenedores/06-almacenamiento/03-gestion_volumenes.md)
- [Volumenes en orquestacion local](03-contenedores/06-almacenamiento/04-volumenes_orquestacion.md)
- [Buenas practicas de almacenamiento](03-contenedores/06-almacenamiento/05-buenas_practicas.md)
- [Copias de seguridad de volumenes](03-contenedores/06-almacenamiento/06-copias_seguridad.md)

### Redes

- [Redes por defecto](03-contenedores/07-redes/01-redes_por_defecto.md)
- [Tipos de redes](03-contenedores/07-redes/02-tipos_redes.md)
- [Redes definidas por el usuario](03-contenedores/07-redes/03-redes_usuario.md)
- [Publicacion de puertos](03-contenedores/07-redes/04-publicacion_puertos.md)
- [Inspeccion y diagnostico de red](03-contenedores/07-redes/05_inspeccion_diagnostico.md)
- [Seguridad de red](03-contenedores/07-redes/06-seguridad_red.md)

### Orquestacion

- [Intro a Docker Compose](03-contenedores/08-orquestacion/01_intro_compose.md)
- [Estructura del archivo de Compose](03-contenedores/08-orquestacion/02-estructura_archivo.md)
- [Comandos esenciales de Compose](03-contenedores/08-orquestacion/03_comandos_esenciales.md)
- [Dependencias y orden de inicio](03-contenedores/08-orquestacion/04_dependencias_inicio.md)
- [Variables de entorno y archivos .env](03-contenedores/08-orquestacion/05_variables_entorno.md)
- [Ejemplo practico: stack completo](03-contenedores/08-orquestacion/06_ejemplo_practico.md)

### Registro

- [Registro de imagenes](03-contenedores/09-registro/01_registry.md)
- [Publicacion de imagenes](03-contenedores/09-registro/02_publicacion_imagen.md)
- [Registros privados y empresariales](03-contenedores/09-registro/03_registros_privados.md)
- [Seguridad en la distribucion](03-contenedores/09-registro/04_seguridad_distribucion.md)
- [Politicas de etiquetado y versionado](03-contenedores/09-registro/05_versionado_etiquetas.md)
- [CI/CD y registries](03-contenedores/09-registro/06_cicd_registros.md)

### Flujo Trabajo

- [Desarrollo local con contenedores](03-contenedores/10-flujo_trabajo/01_desarrollo_local.md)
- [Integracion continua](03-contenedores/10-flujo_trabajo/02_integracion_continua.md)
- [Despliegue en produccion](03-contenedores/10-flujo_trabajo/03_despliegue_produccion.md)
- [Monitoreo y logging](03-contenedores/10-flujo_trabajo/04_monitoreo_logging.md)
- [Seguridad en produccion](03-contenedores/10-flujo_trabajo/05_seguridad_produccion.md)
- [Limites y el siguiente paso](03-contenedores/10-flujo_trabajo/06_limites_siguiente_paso.md)

## Orquestadores

### Introduccion

- [Por que necesitamos orquestacion](04-orquestadores/01-introduccion/01-por_que_necesitamos_orquestacion.md)
- [Que es un orquestador](04-orquestadores/01-introduccion/02-que_es_orquestador.md)
- [Conceptos clave](04-orquestadores/01-introduccion/03-conceptos_clave.md)
- [Casos de uso tipicos](04-orquestadores/01-introduccion/04-casos_uso_tipicos.md)

### Arquitectura Orquestador

- [Componentes del Control Plane](04-orquestadores/02-arquitectura_orquestador/01-componentes_control_plane.md)
- [Componentes del Worker Node](04-orquestadores/02-arquitectura_orquestador/02-componentes_worker.md)
- [Arquitectura Cliente-Servidor](04-orquestadores/02-arquitectura_orquestador/03-arquitectura_cliente_servidor.md)
- [Modelos de Comunicacion](04-orquestadores/02-arquitectura_orquestador/04-modelos_comunicacion.md)

### Gestion Servicios

- [Definicion declarativa de servicios](04-orquestadores/03-gestion_servicios/01-definicion_declarativa.md)
- [Ciclo de vida de un servicio](04-orquestadores/03-gestion_servicios/02-ciclo_vida_servicio.md)
- [Replicas y escalado](04-orquestadores/03-gestion_servicios/03-replicas_escalado.md)
- [Estrategias de actualizacion](04-orquestadores/03-gestion_servicios/04-estrategias_actualizacion.md)
- [Rollback y control de versiones](04-orquestadores/03-gestion_servicios/05-rollback_versiones.md)
- [Tipos de cargas de trabajo](04-orquestadores/03-gestion_servicios/06-tipos_cargas_trabajo.md)

### Service Discovery Redes

- [Modelo de Red en Orquestadores](04-orquestadores/04-service_discovery_redes/01-modelo_red_orquestadores.md)
- [Service Discovery](04-orquestadores/04-service_discovery_redes/02-service_discovery.md)
- [Tipos de Servicios de Red](04-orquestadores/04-service_discovery_redes/03-tipos_servicios_red.md)
- [Ingress y Gateways](04-orquestadores/04-service_discovery_redes/04-ingress_gateways.md)
- [Politicas de Red](04-orquestadores/04-service_discovery_redes/05-politicas_red.md)

### Configuracion Secretos Almacenamiento

- [Gestion de configuracion en orquestadores](04-orquestadores/05-configuracion_secretos_almacenamiento/01-gestion_configuracion.md)
- [Gestion de secretos en orquestadores](04-orquestadores/05-configuracion_secretos_almacenamiento/02-gestion_secretos.md)
- [Almacenamiento persistente en orquestadores](04-orquestadores/05-configuracion_secretos_almacenamiento/03-almacenamiento_persistente.md)
- [Buenas practicas de almacenamiento en orquestadores](04-orquestadores/05-configuracion_secretos_almacenamiento/04-buenas_practicas_almacenamiento.md)

### Escalado Autorrepacion

- [Escalado Horizontal con HPA](04-orquestadores/06-escalado_autorrepacion/01-escalado_horizontal_hpa.md)
- [Escalado Vertical con VPA](04-orquestadores/06-escalado_autorrepacion/02-escalado_vertical_vpa.md)
- [Autorrepacion y Health Checks](04-orquestadores/06-escalado_autorrepacion/03-autorrepacion_self_healing.md)
- [Gestion de Nodos](04-orquestadores/06-escalado_autorrepacion/04-gestion_nodos.md)

### Seguridad Gobernanza

- [Contextos de Seguridad](04-orquestadores/07-seguridad_gobernanza/01-contextos_seguridad.md)
- [RBAC - Control de Acceso Basado en Roles](04-orquestadores/07-seguridad_gobernanza/02-rbac.md)
- [Politicas de Seguridad](04-orquestadores/07-seguridad_gobernanza/03-politicas_seguridad.md)
- [Seguridad de Red](04-orquestadores/07-seguridad_gobernanza/04-seguridad_red.md)
- [Gestion de Imagenes Seguras](04-orquestadores/07-seguridad_gobernanza/05-gestion_imagenes_seguras.md)
- [Limites de Recursos](04-orquestadores/07-seguridad_gobernanza/06-limites_recursos.md)

### Monitoreo Logging Observabilidad

- [Los tres pilares de la observabilidad](04-orquestadores/08-monitoreo_logging_observabilidad/01-tres_pilares_observabilidad.md)
- [Monitoreo del cluster](04-orquestadores/08-monitoreo_logging_observabilidad/02-monitoreo_cluster.md)
- [Gestion centralizada de logs](04-orquestadores/08-monitoreo_logging_observabilidad/03-gestion_centralizada_logs.md)
- [Trazas distribuidas](04-orquestadores/08-monitoreo_logging_observabilidad/04-distributed_tracing.md)
- [Alertas y notificaciones](04-orquestadores/08-monitoreo_logging_observabilidad/05-alertas_notificaciones.md)

### Herramientas Locales Desarrollo

- [Clusteres locales de desarrollo](04-orquestadores/09-herramientas_locales_desarrollo/01-clusteres_locales_desarrollo.md)
- [Entornos gestionados en nube](04-orquestadores/09-herramientas_locales_desarrollo/02-entornos_gestionados_nube.md)
- [Herramientas de desarrollo](04-orquestadores/09-herramientas_locales_desarrollo/03-herramientas_desarrollo.md)

### Buenas Practicas Patrones

- [Diseno de aplicaciones para orquestadores](04-orquestadores/10-buenas_practicas_patrones/01-diseno_aplicaciones_orquestadores.md)
- [Patrones de orquestacion](04-orquestadores/10-buenas_practicas_patrones/02-patrones_orquestacion.md)
- [Gestion de configuraciones complejas](04-orquestadores/10-buenas_practicas_patrones/03-gestion_configuraciones_complejas.md)
- [GitOps](04-orquestadores/10-buenas_practicas_patrones/04-gitops.md)
- [Multi-cluster y multi-nube](04-orquestadores/10-buenas_practicas_patrones/05-multi_cluster_multi_nube.md)

### Casos Uso Decisiones

- [Cuando usar un orquestador](04-orquestadores/11-casos_uso_decisiones/01-cuando_usar_orquestador.md)
- [Comparacion de orquestadores](04-orquestadores/11-casos_uso_decisiones/02-comparacion_orquestadores.md)
- [Migracion a orquestadores](04-orquestadores/11-casos_uso_decisiones/03-migracion_orquestadores.md)
- [Tendencias y futuro de la orquestacion](04-orquestadores/11-casos_uso_decisiones/04-tendencias_futuro.md)

## Infraestructura como código (IaC)

### Fundamentos IaC

- [Que es Infraestructura como Codigo](05-iac/01-fundamentos_iac/01-que_es_iac.md)
- [Enfoques Imperativo y Declarativo en IaC](05-iac/01-fundamentos_iac/02-imperativo_vs_declarativo.md)
- [Beneficios de Infraestructura como Codigo](05-iac/01-fundamentos_iac/03-beneficios_iac.md)

### Introduccion Herramientas

- [Panorama de Herramientas de IaC](05-iac/02-introduccion_herramientas/01-panorama_herramientas.md)
- [Arquitectura de una Herramienta IaC](05-iac/02-introduccion_herramientas/02-arquitectura_iac.md)
- [Conceptos Clave de IaC](05-iac/02-introduccion_herramientas/03-conceptos_clave.md)

### Ciclo Vida Flujo

- [Inicializacion de proyecto](05-iac/03-ciclo_vida_flujo/01-inicializacion_proyecto.md)
- [Escritura de configuracion](05-iac/03-ciclo_vida_flujo/02-escritura_configuracion.md)
- [Planificacion y preview](05-iac/03-ciclo_vida_flujo/03-planificacion_preview.md)
- [Aplicacion y provisioning](05-iac/03-ciclo_vida_flujo/04-aplicacion_provisioning.md)
- [Destruccion y limpieza](05-iac/03-ciclo_vida_flujo/05-destruccion_limpieza.md)

### Lenguajes Sintaxis

- [Tipos de lenguajes de definicion](05-iac/04-lenguajes_sintaxis/01-tipos_lenguajes.md)
- [Gestion de variables](05-iac/04-lenguajes_sintaxis/02-gestion_variables.md)
- [Gestion de salidas](05-iac/04-lenguajes_sintaxis/03-gestion_salidas.md)
- [Consultas de datos externos](05-iac/04-lenguajes_sintaxis/04_consultas_datos_externos.md)

### Gestion Estado

- [Importancia del Estado en IaC](05-iac/05-gestion_estado/01-importancia_estado.md)
- [Almacenamiento Local y Remoto del Estado](05-iac/05-gestion_estado/02_almacenamiento_local_remoto.md)
- [Bloqueo del Estado en IaC](05-iac/05-gestion_estado/03-bloqueo_estado.md)
- [Operaciones Avanzadas con el Estado](05-iac/05-gestion_estado/04-operaciones_avanzadas.md)

### Modularidad Reutilizacion

- [Concepto de Modulo en Infraestructura como Codigo](05-iac/06-modularidad_reutilizacion/01-concepto_modulo.md)
- [Estructura Estandar de un Modulo](05-iac/06-modularidad_reutilizacion/02-estructura_modulo.md)
- [Fuentes de Modulos](05-iac/06-modularidad_reutilizacion/03-fuentes_modulos.md)
- [Buenas Practicas de Modularizacion](05-iac/06-modularidad_reutilizacion/04-buenas_practicas_modularizacion.md)

### Dependencias Ciclo Vida

- [Dependencias implicitas](05-iac/07-dependencias_ciclo_vida/01-dependencias_implicitas.md)
- [Dependencias explicitas](05-iac/07-dependencias_ciclo_vida/02-dependencias_explicitas.md)
- [Grafo de ejecucion](05-iac/07-dependencias_ciclo_vida/03-grafo_ejecucion.md)
- [Lifecycle rules](05-iac/07-dependencias_ciclo_vida/04-lifecycle_rules.md)

### Provisionamiento Configuracion

- [Limite de IaC: Provisionamiento vs Configuracion](05-iac/08-provisionamiento_configuracion/01_limite_iac.md)
- [Mecanismos de Bootstrapping](05-iac/08-provisionamiento_configuracion/02_mecanismos_bootstrapping.md)
- [Integracion con Herramientas de Gestion de Configuracion](05-iac/08-provisionamiento_configuracion/03_herramientas_configuracion.md)
- [Imagenes Inmutables: Eliminar la Configuracion en Runtime](05-iac/08-provisionamiento_configuracion/04_imagenes_inmutables.md)

### Multiples Entornos

- [Estrategias de aislamiento en multiples entornos](05-iac/09-multiples_entornos/01-estrategias_aislamiento.md)
- [Parametrizacion por entorno](05-iac/09-multiples_entornos/02_parametrizacion_entorno.md)

### Automatizacion CI/CD

- [Ejecucion remota de IaC](05-iac/10-automatizacion_cicd/01-ejecucion_remota.md)
- [Integracion con pipelines CI/CD](05-iac/10-automatizacion_cicd/02-integracion_pipelines.md)
- [GitOps para infraestructura](05-iac/10-automatizacion_cicd/03-gitops_infraestructura.md)

### Seguridad Gobernanza

- [Gestion de secretos en IaC](05-iac/11-seguridad_gobernanza/01-gestion_secretos.md)
- [Seguridad del state file](05-iac/11-seguridad_gobernanza/02-seguridad_estado.md)
- [Principio de minimo privilegio](05-iac/11-seguridad_gobernanza/03-minimo_privilegio.md)
- [Policy as Code](05-iac/11-seguridad_gobernanza/04-policy_as_code.md)
- [Pruebas en infraestructura como codigo](05-iac/11-seguridad_gobernanza/05-pruebas_iac.md)

## Monitorización y observabilidad

### Fundamentos

- [Monitoreo vs Observabilidad](06-monitoreo/01-fundamentos/01-monitoreo_vs_observabilidad.md)
- [Los Tres Pilares de la Observabilidad](06-monitoreo/01-fundamentos/02-los_tres_pilares.md)
- [Por que Observabilidad](06-monitoreo/01-fundamentos/03-por_que_observabilidad.md)

### Metricas

- [Tipos de metricas](06-monitoreo/02-metricas/01-tipos_de_metricas.md)
- [Metricas clave](06-monitoreo/02-metricas/02-metricas_clave.md)
- [Coleccion y almacenamiento de metricas](06-monitoreo/02-metricas/03-coleccion_almacenamiento.md)

### Logs

- [Fundamentos de Logging](06-monitoreo/03-logs/01-fundamentos_de_logging.md)
- [Gestion Centralizada de Logs](06-monitoreo/03-logs/02-gestion_centralizada_logs.md)
- [Correlacion de Logs](06-monitoreo/03-logs/03-correlacion_de_logs.md)

### Traces Tracing

- [Que es el tracing](06-monitoreo/04-traces_tracing/01-que_es_el_tracing.md)
- [Instrumentacion](06-monitoreo/04-traces_tracing/02-instrumentacion.md)
- [Analisis de traces](06-monitoreo/04-traces_tracing/03-analisis_de_traces.md)

### Alertas Respuesta

- [Diseno de Alertas](06-monitoreo/05-alertas_respuesta/01-diseno_de_alertas.md)
- [Escalamiento y Respuesta](06-monitoreo/05-alertas_respuesta/02-escalamiento_respuesta.md)
- [SLI, SLO y SLA](06-monitoreo/05-alertas_respuesta/03-sla_slo_sli.md)

### Herramientas Ecosistema

- [Herramientas de Metricas](06-monitoreo/06-herramientas_ecosistema/01-herramientas_de_metricas.md)
- [Herramientas de Logs](06-monitoreo/06-herramientas_ecosistema/02-herramientas_de_logs.md)
- [Herramientas de Tracing](06-monitoreo/06-herramientas_ecosistema/03-herramientas_de_tracing.md)
- [Plataformas de Observabilidad Completa](06-monitoreo/06-herramientas_ecosistema/04-plataformas_observabilidad_completa.md)

## Integración y despliegue continuos (CI/CD)

### Fundamentos

- [Que es CI/CD](07-cicd/01-fundamentos/01-que_es_cicd.md)
- [Por que CI/CD](07-cicd/01-fundamentos/02-por_que_cicd.md)
- [El Pipeline](07-cicd/01-fundamentos/03-el_pipeline.md)

### Integracion Continua

- [Build Automatizado](07-cicd/02-integracion_continua/01-build_automatizado.md)
- [Testing Automatizado](07-cicd/02-integracion_continua/02-testing_automatizado.md)
- [Ramas y Fusion](07-cicd/02-integracion_continua/03-ramas_y_fusion.md)

### Despliegue Continuo

- [Estrategias de Despliegue](07-cicd/03-despliegue_continuo/01-estrategias_de_despliegue.md)
- [Gestion de Entornos](07-cicd/03-despliegue_continuo/02-gestion_de_entornos.md)
- [Rollback y Recuperacion](07-cicd/03-despliegue_continuo/03-rollback_y_recuperacion.md)

### Herramientas

- [Herramientas de Integracion Continua](07-cicd/04-herramientas/01-herramientas_ci.md)
- [Herramientas de Despliegue y Despliegue Continuo](07-cicd/04-herramientas/02-herramientas_deployment.md)
- [Repositorios de Artefactos](07-cicd/04-herramientas/03-artifact_repositories.md)

### Seguridad CI/CD

- [Seguridad en Pipelines CI/CD](07-cicd/05-seguridad_cicd/01-seguridad_en_pipelines.md)
- [Seguridad de la Cadena de Suministro](07-cicd/05-seguridad_cicd/02-supply_chain_security.md)

## Servidores

### Introduccion Servidores

- [Que es un servidor](08-servidores/01-introduccion_servidores/01-que_es_un_servidor.md)
- [Tipos comunes de servidores](08-servidores/01-introduccion_servidores/02-tipos_comunes.md)
- [Servidor vs estacion de trabajo](08-servidores/01-introduccion_servidores/03-servidor_vs_estacion.md)
- [Modelos de implementacion](08-servidores/01-introduccion_servidores/04-modelos_implementacion.md)
- [Conceptos clave](08-servidores/01-introduccion_servidores/05-conceptos_clave.md)
- [Herramientas de acceso remoto](08-servidores/01-introduccion_servidores/06-herramientas_acceso_remoto.md)

### Preparacion SO

- [Eleccion del sistema operativo](08-servidores/02-preparacion_so/01-eleccion_so.md)
- [Instalacion minima](08-servidores/02-preparacion_so/02-instalacion_minima.md)
- [Configuracion inicial](08-servidores/02-preparacion_so/03-configuracion_inicial.md)
- [Usuarios y privilegios](08-servidores/02-preparacion_so/04-usuarios_privilegios.md)
- [Configuracion de red](08-servidores/02-preparacion_so/05-configuracion_red.md)
- [Herramientas esenciales](08-servidores/02-preparacion_so/06-herramientas_esenciales.md)

### Gestion Servicios Procesos

- [systemd: el sistema de init moderno](08-servidores/03-gestion_servicios_procesos/01-systemd.md)
- [Servicios comunes en servidores Linux](08-servidores/03-gestion_servicios_procesos/02-servicios_comunes.md)
- [Gestion de procesos](08-servidores/03-gestion_servicios_procesos/03-gestion_procesos.md)
- [Tareas programadas](08-servidores/03-gestion_servicios_procesos/04-tareas_programadas.md)
- [Logs del sistema](08-servidores/03-gestion_servicios_procesos/05-logs_sistema.md)
- [Dependencias y conflictos entre servicios](08-servidores/03-gestion_servicios_procesos/06-dependencias_conflictos.md)

### Seguridad Basica

- [Firewall](08-servidores/04-seguridad_basica/01-firewall.md)
- [SSH seguro](08-servidores/04-seguridad_basica/02-ssh_seguro.md)
- [Actualizaciones automaticas](08-servidores/04-seguridad_basica/03-actualizaciones_automaticas.md)
- [Gestion de usuarios y permisos](08-servidores/04-seguridad_basica/04-usuarios_permisos.md)
- [Herramientas de seguridad](08-servidores/04-seguridad_basica/05-herramientas_seguridad.md)
- [Auditoria basica de accesos](08-servidores/04-seguridad_basica/06-auditoria_basica.md)

### Automatizacion Despliegue

- [Scripts Shell para Automatizacion de Despliegue](08-servidores/05-automatizacion_despliegue/01-scripts_shell.md)
- [Gestion de Configuracion](08-servidores/05-automatizacion_despliegue/02-gestion_configuracion.md)
- [Ansible Basico](08-servidores/05-automatizacion_despliegue/03-ansible_basico.md)
- [Infraestructura como Codigo](08-servidores/05-automatizacion_despliegue/04-iac.md)
- [CI/CD Integrado](08-servidores/05-automatizacion_despliegue/05-cicd_integrado.md)
- [Buenas Practicas en Automatizacion de Despliegue](08-servidores/05-automatizacion_despliegue/06-buenas_practicas.md)

### Monitoreo Mantenimiento

- [Monitoreo basico de servidores](08-servidores/06-monitoreo_mantenimiento/01-monitoreo_basico.md)
- [Herramientas de monitoreo](08-servidores/06-monitoreo_mantenimiento/02-herramientas_monitoreo.md)
- [Logs y alertas](08-servidores/06-monitoreo_mantenimiento/03-logs_alertas.md)
- [Copias de seguridad](08-servidores/06-monitoreo_mantenimiento/04-copias_seguridad.md)
- [Mantenimiento rutinario](08-servidores/06-monitoreo_mantenimiento/05-mantenimiento_rutinario.md)
- [Actualizacion con cuidado](08-servidores/06-monitoreo_mantenimiento/06-actualizacion_con_cuidado.md)

### Servidores Nube

- [Modelo IaaS](08-servidores/07-servidores_nube/01-modelo_iaas.md)
- [Provisionamiento de instancias](08-servidores/07-servidores_nube/02-provisionamiento.md)
- [Red y seguridad en la nube](08-servidores/07-servidores_nube/03-red_seguridad_nube.md)
- [Acceso al servidor](08-servidores/07-servidores_nube/04-acceso_servidor.md)
- [Almacenamiento en la nube](08-servidores/07-servidores_nube/05-almacenamiento_nube.md)
- [Ventajas y desafios de la nube](08-servidores/07-servidores_nube/06-ventajas_desafios.md)

### Buenas Practicas

- [Documentacion en Infraestructura](08-servidores/08-buenas_practicas/01-documentacion.md)
- [Principios de Diseno en Infraestructura](08-servidores/08-buenas_practicas/02-principios_diseno.md)
- [Gestion de Secretos](08-servidores/08-buenas_practicas/03-gestion_secretos.md)
- [Pruebas y Entornos](08-servidores/08-buenas_practicas/04-pruebas_entornos.md)
- [Cumplimiento y Etica](08-servidores/08-buenas_practicas/05-cumplimiento_etica.md)
- [Cultura DevOps](08-servidores/08-buenas_practicas/06-cultura_devops.md)

