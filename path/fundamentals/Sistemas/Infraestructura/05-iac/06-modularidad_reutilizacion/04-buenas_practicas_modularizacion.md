# Buenas Practicas de Modularizacion

La modularizacion sin disciplina genera mas problemas de los que resuelve: modulos que hacen demasiado, modulos que hacen demasiado poco, interfaces inestables que rompen consumidores en cada actualizacion, y una proliferation de codigo muerto que nadie se atrevera a eliminar. Las practicas consagradas en la comunidad Terraform establecen principios claros para evitar estos problemas y construir una base de modulos que sea genuinamente reutilizable y mantenible a largo plazo.

El objetivo fundamental es que cada modulo represente una decision de diseno concreta, testable y versionable. Un modulo que combina responsabilidades disparate —una base de datos, un balanceador de carga y un bucket S3 en la misma unidad— viola el principio de responsabilidad unica y hace imposible reutilizar cualquiera de esas partes de forma independiente. La modularizacion efectiva es, en esencia, la aplicacion rigurosa de separacion de responsabilidades al dominio de la infraestructura.

## Principio de Responsabilidad Unica

Cada modulo debe tener una y solo una razon para cambiar. Esto se traduce en una pregunta concreta: si el modulo gestiona multiples tipos de recursos, es posible que uno de ellos necesite actualizar su configuracion sin afectar a los otros? Si la respuesta es si, el modulo tiene demasiadas responsabilidades:

```hcl
# MAL: modulo que hace demasiado
module "todo_en_uno" {
  source = "./modules/mega-stack"

  vpc_cidr         = "10.0.0.0/16"
  db_engine        = "postgres"
  db_instance_class = "db.t3.medium"
  enable_cdn       = true
  domain_name      = "ejemplo.com"
}
```

```hcl
# BIEN: responsabilidad unica
module "vpc" {
  source = "./modules/vpc"
  cidr   = "10.0.0.0/16"
}

module "database" {
  source         = "./modules/rds-postgres"
  vpc_id         = module.vpc.id
  subnet_ids     = module.vpc.database_subnet_ids
  instance_class = "db.t3.medium"
}

module "cdn" {
  source      = "./modules/cloudfront"
  domain_name = "ejemplo.com"
  origin_dns  = module.load_balancer.dns_name
}
```

Cada modulo puede versionarse, testearse y actualizarse independientemente. Una actualizacion del motor de base de datos no requiere tocar la configuracion de la CDN, y viceversa.

El patron inverso —modulos demasiado pequenos— tambien es problematico. Un modulo que contiene un unico recurso sin logica adicional agrega complejidad sin valor: la abstraction tiene un costo en indirecion que solo se justifica si el modulo encapsula decisiones de configuracion multiples o logicas condicionales.

> Un buen umbral de referencia: un modulo debe contener entre 3 y 15 recursos. Menos de 3 sugiere que la abstraccion es innecesaria; mas de 15 sugiere que debe descomponerse en submodulos.

## Versionado Semantico

El versionado semantico (semver) aplica el estandar `MAJOR.MINOR.PATCH` a los modulos de infraestructura con reglas adaptadas al dominio:

- **MAJOR**: Cambios que rompen compatibilidad hacia atras — eliminar un output, renombrar una variable obligatoria, cambiar el tipo de un recurso gestionado.
- **MINOR**: Nuevas funcionalidades compatibles — agregar una variable opcional con valor por defecto, crear un output adicional, habilitar un recurso auxiliar opcional.
- **PATCH**: Correcciones que no alteran la interfaz — corregir una politica de seguridad, actualizar la AMI por defecto, resolver un bug en una validacion.

```hcl
# Registro de versiones en el modulo
# CHANGELOG.md

## [3.2.1] - 2026-07-15
### Fixed
- Corregida politica de security group que permitia acceso SSH desde 0.0.0.0/0

## [3.2.0] - 2026-07-01
### Added
- Variable `enable_detailed_monitoring` (opcional, default: false)
- Output `instance_role_arn`

## [3.0.0] - 2026-06-01
### Changed (BREAKING)
- Variable `subnet_id` renombrada a `subnet_ids` (lista)
- Requiere Terraform >= 1.5.0
```

Los consumidores especifican rangos de version con la sintaxis `~>` para protegerse de cambios incompatibles:

```hcl
module "servidor" {
  source  = "git::https://github.com/org/modules.git//ec2-instance"
  version = "~> 3.2"   #acepta 3.2.x, 3.3.x, pero no 4.0.0
}
```

> Antes de incrementar una version MAJOR, evaluar si el cambio puede implementarse como una variable opcional nueva con valor por defecto que preserve el comportamiento anterior. Muchos cambios "rotos" pueden evitarse mediante un diseno mas cuidadoso de la interfaz.

## Composicion: Construccion desde Modulos Pequenos

La composicion es el patron que conecta modulos individuales en sistemas completos. Un modulo de "aplicacion completa" no debe contener la logica de cada componente —debe orquestar la invocacion de modulos especializados y conectar sus outputs con sus inputs:

```hcl
# infrastructure/main.tf

module "networking" {
  source = "./modules/vpc"

  name             = "produccion"
  cidr             = "10.0.0.0/16"
  availability_zones = ["eu-west-1a", "eu-west-1b", "eu-west-1c"]
  
  private_subnet_cidrs = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  public_subnet_cidrs  = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]
  
  enable_nat_gateway = true
}

module "database" {
  source = "./modules/rds-postgres"

  name               = "app-db"
  vpc_id             = module.networking.vpc_id
  subnet_ids         = module.networking.database_subnet_ids
  
  instance_class     = "db.r6g.large"
  allocated_storage  = 100
  multi_az           = true
  
  backup_retention_days = 7
}

module "app_cluster" {
  source = "./modules/ecs-fargate"

  name            = "app-api"
  vpc_id          = module.networking.vpc_id
  subnet_ids      = module.networking.private_subnet_ids
  
  container_image = "123456789.dkr.ecr.eu-west-1.amazonaws.com/api:v2.1.0"
  container_port  = 8080
  
  desired_count    = 3
  cpu              = 512
  memory           = 1024
  
  environment = {
    DATABASE_URL = module.database.connection_string
    DB_HOST      = module.database.endpoint
    DB_PORT      = module.database.port
  }
  
  security_group_rules = {
    alb_ingress = {
      type                     = "ingress"
      from_port                = 8080
      to_port                  = 8080
      source_security_group_id = module.load_balancer.security_group_id
    }
    db_egress = {
      type              = "egress"
      to_port           = 5432
      security_group_id = module.database.security_group_id
    }
  }
}

module "load_balancer" {
  source = "./modules/alb"

  name               = "app-alb"
  vpc_id             = module.networking.vpc_id
  subnet_ids         = module.networking.public_subnet_ids
  
  certificate_arn    = module.acm.certificate_arn
  health_check_path  = "/health"
  
  targets = {
    app = {
      port             = 8080
      target_group_arn = module.app_cluster.target_group_arn
    }
  }
}
```

Este ejemplo demuestra como la composicion crea un sistema completo a partir de modulos especializados. Cada modulo contiene la complejidad de su dominio —los detalles de configuracion de RDS son distintos a los de ECS Fargate— pero la configuracion orquestadora es legible y mantenible porque solo muestra las conexiones entre componentes, no los detalles de implementacion de cada uno.

## Testing de Modulos

Un modulo sin tests es un modulo cuyo correcto funcionamiento depende de la fe. Existen tres niveles de testing en Terraform:

**Validacion de variables**: restricciones declarativas en `variables.tf` que interceptan valores invalidos antes de cualquier ejecucion.

**Plan testing**: verifica que el plan de ejecucion produce los recursos esperados sin efectivamente crearlos. Es rapido y seguro para ejecutarse en CI:

```hcl
# test/plan_test.go (usando Terratest o Terraform test)

run "plan_produces_expected_resources" {
  command = plan

  module {
    source = "./modules/ec2-instance"
  }

  variables {
    name          = "test-instance"
    instance_type = "t3.micro"
    vpc_id        = "vpc-test12345"
  }

  assert {
    condition     = length(output.instance_id) > 0
    error_message = "El instance_id no debe estar vacio"
  }
}
```

**Integration testing**: crea infraestructura real en un entorno temporal, verifica su funcionamiento, y la destruye al finalizar. Es el nivel mas confiable pero tambien el mas costoso en tiempo y recursos:

```hcl
run "database_connects_successfully" {
  command = apply

  module {
    source = "./modules/rds-postgres"
  }

  variables {
    name           = "test-db"
    vpc_id         = module.test_vpc.id
    subnet_ids     = module.test_vpc.private_subnet_ids
    instance_class = "db.t3.micro"
  }

  # Verificar que el endpoint es accesible
  assert {
    condition     = can(regex(".*\\.rds\\.amazonaws\\.com$", output.endpoint))
    error_message = "El endpoint debe ser un dominio RDS valido"
  }
}
```

> Ejecute tests de plan en cada pull request y tests de integracion en merges a la rama principal. Esta combinacion equilibra velocidad de feedback con confiabilidad.

## Gobernanza de Modulos en Organizaciones

A medida que la cantidad de modulos crece, se necesitan mecanismos para controlar calidad, promover adopcion y evitar la proliferacion de soluciones duplicadas. Las estrategias efectivas incluyen:

- **Catalogo interno**: un repositorio central con los modulos aprobados, documentados y versionados. Los equipos consumen del catalogo pero no publican directamente — un equipo de plataforma revisa y aprueba las contribuciones.
- **Modulos de obligatorio uso**: determinados modulos —como los de conformidad de seguridad o logging— se exigen en todas las configuraciones mediante politicas OPA/Sentinel.
- **Deprecacion gradual**: los modulos obsoletos se marcan como deprecated con un link al modulo que lo reemplaza, pero no se eliminan durante al menos dos versiones menores para dar tiempo a la migracion.

```hcl
# Politica de OPA que obliga el uso del modulo de logging aprobado

deny[msg] {
  resource := input.plan.resource_changes[_]
  resource.type == "aws_cloudwatch_log_group"
  not startswith(resource.address, "module.app_logging.")
  msg := sprintf(
    "Los grupos de logs deben crearse via module.app_logging. Resource: %s",
    [resource.address]
  )
}
```

La gobernanza no debe ser un obstaculo —debe ser un enabler que garantice que los modulos que circulan en la organizacion cumplen estandares minimos de calidad, seguridad y documentacion.

