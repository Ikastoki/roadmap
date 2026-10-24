# Que es Terraform

Infraestructura como Codigo no seria la disciplina que es hoy sin una herramienta que encapsulara la complejidad de orquestar multiples nubes con una sintaxis unificada. Terraform, creado por HashiCorp en 2014, es esa herramienta: un binario escrito en Go que actua como un plano ejecutable para la infraestructura. Donde antes cada plataforma (AWS, Azure, GCP) requeria su propio conjunto de scripts, CLI y formatos, Terraform introduce una capa de abstraccion que permite describir el estado final deseado de toda la infraestructura --redes, servidores, balanceadores, bases de datos-- en un unico lenguaje declarativo llamado HCL.

## Arquitectura de Terraform

La arquitectura de Terraform se compone de cuatro componentes fundamentales que interactuan entre si:

**Core**: el binario principal de Terraform. Es el motor que lee los archivos de configuracion (`.tf`), construye el grafico de dependencias entre recursos, calcula el plan de ejecucion y aplica los cambios. No contiene conocimiento especifico de ningun proveedor cloud; delega esa responsabilidad a los plugins. El core implementa la logica de alto nivel: interpretar HCL, resolver referencias, determinar el orden optimo de operaciones y gestionar el ciclo de vida del estado.

**Providers**: plugins que actuan como puentes entre Terraform y las APIs de los servicios. Cada proveedor sabe como traducir los bloques de configuracion en llamadas API concretas: autenticacion, creacion, lectura, actualizacion y eliminacion de recursos. Cuando ejecutas `terraform init`, el core descarga los providers declarados en el archivo de configuracion desde el Registry. Los providers pueden ser oficiales (mantenidos por HashiCorp o por el propio proveedor cloud), comunitarios o desarrollados a medida.

**State**: el archivo `terraform.tfstate` es el corazon del modelo de Terraform. Contiene el mapeo exacto entre los recursos declarados en codigo y los recursos reales existentes en los proveedores. Cada vez que Terraform ejecuta un plan, compara el estado actual contra la configuracion para determinar las diferencias. Sin state, Terraform no podria saber que recursos ya existen, cuales deben modificarse o cuales deben destruirse. El state tambien almacena metadatos de dependencias y, potencialmente, valores sensibles.

**Registry**: el repositorio publico en `registry.terraform.io` donde se alojan providers y modulos. Funciona como un mercado de componentes reutilizables: desde el provider oficial de AWS hasta modulos comunitarios para desplegar un cluster de Kubernetes completo. Cada provider y modulo en el Registry sigue un sistema de versionado semantico que permite a los equipos fijar versiones especificas y evitar roturas por cambios inesperados.

## HCL (HashiCorp Configuration Language)

HCL no es solo un formato de configuracion; es un lenguaje disenado especificamente para describir infraestructura de forma legible tanto para humanos como para maquinas. Su sintaxis combina bloques, atributos y expresiones con una estructura que resulta familiar a cualquiera que haya trabajado con JSON o YAML, pero anadiendo capacidades de programacion como interpolacion, condicionales, bucles y funciones integradas.

```hcl
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"

  tags = {
    Name = "ServidorWeb"
  }
}
```

Cada bloque en HCL tiene una estructura de tres partes: el tipo de bloque (`resource`), una etiqueta que identifica el proposito (`"aws_instance"`) y una segunda etiqueta que actua como identificador local (`"web"`). Los atributos dentro del bloque siguen la forma `clave = valor`, con tipos que incluyen strings, numeros, booleanos, listas, mapas, objetos y tuplas.

La interpolacion permite referenciar valores de otros recursos, variables o funciones: `subnet_id = aws_subnet.principal.id`. Terraform resuelve estas referencias construyendo un grafico de dependencias que determina el orden de creacion.

## HashiCorp y el ecosistema

HashiCorp construyo Terraform como parte de una vision mas amplia de automatizacion de infraestructura que incluye otras herramientas como Vault (gestion de secretos), Consul (descubrimiento de servicios) y Nomad (orquestacion de tareas). Sin embargo, Terraform se ha convertido en el producto estrella de la compania y en el estandar de facto para IaC multiplataforma. Su adopcion masiva se debe a que resuelve un problema universal: cualquier equipo que gestione infraestructura en la nube necesita una forma reproducible, versionable y colaborativa de hacerlo, y Terraform ofrece exactamente eso con una curva de aprendizaje razonable y un ecosistema de proveedores que cubre practicamente cualquier servicio cloud moderno.

