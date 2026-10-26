# Infraestructura como Codigo

Configurar un servidor manualmente implica ejecutar comandos en un terminal: crear una instancia en el cloud, asignarle una IP, instalar el sistema operativo, configurar la red, instalar dependencias. Cada paso es una decision humana que queda en la memoria del administrador y nowhere mas. Si el servidor falla y debe recrearse, el administrador repite los pasos desde la memoria, con el riesgo de olvidar algo o hacerlo de forma ligeramente distinta. Infraestructura como Codigo (IaC) elimina esta fragilidad al representar toda la infraestructura como archivos de texto versionados en un repositorio. Los servidores, redes, balanceadores de carga y reglas de firewall se declaran en codigo que puede revisarse, aprobarse, probarse y ejecutarse repetidamente con resultados deterministas.

## La separacion de responsabilidades: provision vs configuracion

Cuando se habla de IaC, es fundamental distinguir entre dos fases que corresponden a problemas diferentes y requieren herramientas distintas.

### Provisioning: crear la infraestructura

La provision es el proceso de crear los recursos fisicos o virtuales: instancias de servidor, redes virtuales, grupos de seguridad, discos, balanceadores de carga. Terraform es la herramienta de referencia para esta fase. Terraform describe los recursos que deben existir y las relaciones entre ellos. Al ejecutarse, compara el estado actual con el estado declarado y crea, modifica o elimina recursos para alcanzar el estado deseado.

```hcl
resource "aws_instance" "web" {
  count         = 3
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"

  tags = {
    Name = "web-${count.index + 1}"
  }
}
```

Este bloque Terraform crea tres instancias EC2 en AWS. La palabra clave `count` genera multiples instancias desde un unico bloque de recurso. Si se cambia `count` a 5, Terraform creara dos instancias adicionales. Si se elimina el bloque, Terraform destruira las tres. Terraform no sabe nada sobre que software se ejecuta en esas instancias; solo garantiza que existan.

### Configuracion: preparar el servidor

Una vez que la instancia existe (provisionada), necesita configurarse: instalar nginx, configurar bases de datos, crear usuarios, desplegar la aplicacion. Esta es la responsabilidad de herramientas como Ansible, Puppet o Chef. Estas herramientas asumen que el servidor ya existe y es accesible, y se encargan de llevarlo al estado deseado.

```yaml
---
- name: Configurar servidores web provisionados
  hosts: webservers
  become: yes

  tasks:
    - name: Actualizar sistema
      ansible.builtin.apt:
        upgrade: dist
        update_cache: yes

    - name: Instalar nginx
      ansible.builtin.apt:
        name: nginx
        state: present

    - name: Desplegar aplicacion
      ansible.builtin.git:
        repo: https://github.com/miorg/miapp.git
        dest: /var/www/miapp
        version: main
```

Este playbook asume que los servidores ya existen (fueron provisionados por Terraform o por otro medio) y se encarga de configurarlos. La separacion es clara: Terraordena **que** existe, Ansible ordena **como** esta configurado.

## Por que separar provisioning y configuracion

La separacion no es una preferencia estetica, sino una necesidad tecnica. Las herramientas de provision como Terraform gestionan el ciclo de vida de los recursos: creacion, actualizacion y destruccion. Mantienen un **state file** que registra que recursos existen y sus atributos. Si Terraform y Ansible intentaran gestionar los mismos recursos, sus estados se conflicitarian: Terraform crearia un servidor y Ansible intentaria gestionar los mismos atributos, generando inconsistencias.

Ademas, la separacion permite reutilizacion. Un playbook de Ansible puede configurar servidores provisionados en AWS, en un datacenter fisico o en un portatil local. Un modulo Terraform de AWS puede crear servidores que seran configurados por Ansible, Puppet o cualquier otra herramienta. Las capas no estan acopladas.

> En la practica, Terraform puede invocar Ansible como parte de su ciclo de ejecucion usando el recurso `local-exec` o el provisioner `remote-exec`. Sin embargo, la recomendacion general es mantener las herramientas separadas y orquestarlas desde un pipeline CI/CD que ejecute Terraform primero y Ansible despues.

## Terraform: estado y ciclo de vida

Terraform no solo crea recursos; gestiona su estado a lo largo del tiempo. Cuando se ejecuta `terraform plan`, analiza los archivos `.tf` y compara con el state file para determinar que cambios serian necesarios. Cuando se ejecuta `terraform apply`, aplica esos cambios. Esto permite tres operaciones criticas:

- **Creacion**: recursos que existen en el codigo pero no en la nube se crean.
- **Actualizacion**: recursos que cambiaron en el codigo se modifican.
- **Destruccion**: recursos que existen en la nube pero no en el codigo se eliminan.

El state file debe almacenarse de forma remota y con bloqueo (S3 con DynamoDB, por ejemplo) para que multiples personas puedan trabajar sin conflictos. Compartir el state file entre usuarios sin bloqueo es una de las causas mas comunes de problemas en equipos que adoptan Terraform.

## Combinacion en la practica

Un flujo de trabajo completo de despliegue combinaria ambas herramientas:

1. **Terraform** crea la infraestructura: 3 instancias EC2, un balanceador de carga, un grupo de seguridad.
2. **Ansible** configura los servidores: instala dependencias, despliega la aplicacion, configura nginx.
3. **Terraform** gestiona los recursos a largo plazo: escalar instancias, cambiar tipos, eliminar las que no se necesiten.

Cada paso se ejecuta desde un repositorio Git, con revision de codigo y aprobacion antes de aplicarse en produccion. Esto garantiza que cada cambio en la infraestructura es visible, rastreable y reversible.

