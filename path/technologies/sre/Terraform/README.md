# Terraform

## Índice

- [Fundamentos de IaC](#fundamentos-de-iac)
- [Introducción a Terraform](#introducción-a-terraform)
- [Flujo de trabajo](#flujo-de-trabajo)
- [HCL](#hcl)
- [Estado y backend](#estado-y-backend)
- [Módulos](#módulos)
- [Dependencias](#dependencias)
- [Provisionamiento](#provisionamiento)
- [Workspaces](#workspaces)
- [Terraform Cloud y CI/CD](#terraform-cloud-y-cicd)
- [Seguridad y buenas prácticas](#seguridad-y-buenas-prácticas)

> [!WARNING]
> Los apuntes no tienen acentos ni ñ porque, al probar una extensión nueva con IA, los ha borrado. Cuando termine de subir todo, lo solucionaré.

## Fundamentos de IaC

- [Qué es Infrastructure as Code](01-fundamentos_iac/01-que_es_iac.md)
- [Beneficios de IaC](01-fundamentos_iac/02-beneficios_iac.md)

## Introducción a Terraform

- [Qué es Terraform](02-introduccion/01-que_es_terraform.md)
- [Conceptos clave](02-introduccion/02-conceptos_clave.md)
- [Proveedores y Registry](02-introduccion/03-proveedores_y_registry.md)

## Flujo de trabajo

- [Inicialización](03-flujo_trabajo/01-inicializacion.md)
- [Configuración](03-flujo_trabajo/02-configuracion.md)
- [Plan y apply](03-flujo_trabajo/03-plan_y_apply.md)
- [Destroy](03-flujo_trabajo/04-destroy.md)

## HCL

- [Sintaxis básica](04-hcl/01-sintaxis_basica.md)
- [Variables](04-hcl/02-variables.md)
- [Outputs](04-hcl/03-outputs.md)
- [Data sources](04-hcl/04-data_sources.md)

## Estado y backend

- [Importancia del estado](05-estado_backend/01-importancia_estado.md)
- [Backend local y remoto](05-estado_backend/02-backend_local_remoto.md)
- [Comandos de estado](05-estado_backend/03-comandos_estado.md)

## Módulos

- [Concepto de módulos](06-modulos/01-concepto_modulos.md)
- [Uso de módulos](06-modulos/02-uso_de_modulos.md)
- [Buenas prácticas con módulos](06-modulos/03-buenas_practicas.md)

## Dependencias

- [Dependencias implícitas y explícitas](07-dependencias/01-dependencias_implicitas_explicitas.md)
- [Gráfico de dependencias](07-dependencias/02-grafico_dependencias.md)
- [Ciclo de vida](07-dependencias/03-ciclo_de_vida.md)

## Provisionamiento

- [User data y cloud-init](08-provisionamiento/01-user_data_cloudinit.md)
- [Provisioners](08-provisionamiento/02-provisioners.md)
- [Integración con Ansible](08-provisionamiento/03-integracion_ansible.md)

## Workspaces

- [Concepto de workspaces](09-workspaces/01-concepto_workspaces.md)
- [Estrategias multi-entorno](09-workspaces/02-estrategias_multi_entorno.md)
- [Variables por entorno](09-workspaces/03-variables_por_entorno.md)

## Terraform Cloud y CI/CD

- [Terraform Cloud](10-terraform_cloud_cicd/01-terraform_cloud.md)
- [Integración con CI/CD](10-terraform_cloud_cicd/02-integracion_cicd.md)
- [Pipelines y buenas prácticas](10-terraform_cloud_cicd/03-pipelines_buenas_practicas.md)

## Seguridad y buenas prácticas

- [Gestión de secretos](11-seguridad_buenas_practicas/01-gestion_secretos.md)
- [Estado remoto seguro](11-seguridad_buenas_practicas/02-state_remoto_seguro.md)
- [Mínimo privilegio](11-seguridad_buenas_practicas/03-minimo_privilegio.md)
- [Validación y testing](11-seguridad_buenas_practicas/04-validacion_testing.md)
