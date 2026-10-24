# Kubernetes

## Índice

- [Introducción a la orquestación](#introducción-a-la-orquestación)
- [Arquitectura](#arquitectura)
- [kubectl](#kubectl)
- [Pods y Deployments](#pods-y-deployments)
- [Configuración y almacenamiento](#configuración-y-almacenamiento)
- [Redes](#redes)
- [Escalado y gestión avanzada](#escalado-y-gestión-avanzada)
- [Entornos locales y cloud](#entornos-locales-y-cloud)
- [Seguridad y buenas prácticas](#seguridad-y-buenas-prácticas)

> [!WARNING]
> Los apuntes no tienen acentos ni ñ porque, al probar una extensión nueva con IA, los ha borrado. Cuando termine de subir todo, lo solucionaré.

## Introducción a la orquestación

- [Por qué usar orquestación](01-introduccion_orquestacion/01-por_que_orquestacion.md)
- [Qué es Kubernetes](01-introduccion_orquestacion/02-que_es_kubernetes.md)
- [Conceptos clave](01-introduccion_orquestacion/03-conceptos_clave.md)

## Arquitectura

- [Control plane](02-arquitectura/01-control_plane.md)
- [Nodos workers](02-arquitectura/02-nodos_workers.md)
- [Arquitectura cliente-servidor](02-arquitectura/03-arquitectura_cliente_servidor.md)

## kubectl

- [Instalación y configuración](03-kubectl/01-instalacion_configuracion.md)
- [Comandos esenciales](03-kubectl/02-comandos_esenciales.md)
- [Contextos y namespaces](03-kubectl/03-contextos_namespaces.md)

## Pods y Deployments

- [Creación de Pods](04-pods_y_deployments/01-creacion_de_pods.md)
- [Deployments](04-pods_y_deployments/02-deployments.md)
- [Services](04-pods_y_deployments/03-services.md)
- [Health checks](04-pods_y_deployments/04-health_checks.md)

## Configuración y almacenamiento

- [ConfigMaps](05-configuracion_almacenamiento/01-configmaps.md)
- [Secrets](05-configuracion_almacenamiento/02-secrets.md)
- [Volúmenes](05-configuracion_almacenamiento/03-volumenes.md)

## Redes

- [Modelo de red](06-redes/01-modelo_de_red.md)
- [Services y DNS](06-redes/02-services_dns.md)
- [Ingress](06-redes/03-ingress.md)
- [Network Policies](06-redes/04-network_policies.md)

## Escalado y gestión avanzada

- [Horizontal Pod Autoscaler](07-escalado_gestion_avanzada/01-hpa.md)
- [Vertical Pod Autoscaler](07-escalado_gestion_avanzada/02-vpa.md)
- [Rolling updates](07-escalado_gestion_avanzada/03-rolling_updates.md)
- [DaemonSets y StatefulSets](07-escalado_gestion_avanzada/04-daemonsets_statefulsets.md)

## Entornos locales y cloud

- [Minikube](08-entornos_locales_cloud/01-minikube.md)
- [kind](08-entornos_locales_cloud/02-kind.md)
- [Servicios gestionados](08-entornos_locales_cloud/03-servicios_gestionados.md)

## Seguridad y buenas prácticas

- [Security Context](09-seguridad_buenas_practicas/01-security_context.md)
- [RBAC](09-seguridad_buenas_practicas/02-rbac.md)
- [Pod Security Standards](09-seguridad_buenas_practicas/03-pod_security_standards.md)
- [Límites de recursos](09-seguridad_buenas_practicas/04-resource_limits.md)
