# Panorama de Herramientas de IaC

El ecosistema de Infraestructura como Codigo no se reduce a una unica solucion; cada herramienta nacio para resolver un problema especifico dentro del ciclo de vida de la infraestructura. Distinguir entre orquestadores, herramientas cloud-nativas y motores de configuracion es fundamental antes de elegir una pila tecnologica, porque cada categoria asume un modelo de responsabilidad distinto: unas gestionan recursos en la nube desde una abstraction agnostica, otras operan directamente sobre el estado interno de los servidores, y las ultimas estan integradas de forma nativa con un proveedor concreto. Comprender este panorama evita el error comun de intentar resolver problemas de configuracion de SO con un orquestador de nube, o viceversa.

## Herramientas de Orquestacion Multi-Cloud

Las herramientas de orquestacion actuan como capas de abstraction sobre las APIs de los proveedores. Su principal valor es la agnosticidad: el mismo codigo puede desplegar infraestructura en AWS, Azure o GCP sin modificar la logica central. Utilizan un lenguaje declarativo o imperativo y gestionan un estado que mapea la configuracion deseada con los recursos reales.

```hcl
# Ejemplo Terraform - definicion de un recurso multi-cloud
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"

  tags = {
    Name = "web-server"
  }
}
```

```yaml
# Ejemplo Pulumi (TypeScript/infra-estructura como codigo con lenguaje general)
import * as aws from "@pulumi/aws";

const server = new aws.ec2.Instance("web", {
    ami: "ami-0c55b159cbfafe1f0",
    instanceType: "t3.micro",
    tags: { Name: "web-server" },
});
```

```yaml
# Ejemplo Crossplane (Kubernetes-native IaC)
apiVersion: ec2.aws.crossplane.io/v1beta1
kind: Instance
metadata:
  name: web-server
spec:
  forProvider:
    region: us-east-1
    instanceType: t3.micro
    imageId: ami-0c55b159cbfafe1f0
```

| Herramienta   | Lenguaje          | Estado           | Enfoque principal                  |
|---------------|-------------------|------------------|-------------------------------------|
| Terraform     | HCL               | Archivo remoto   | Orquestacion multi-cloud            |
| OpenTofu      | HCL               | Archivo remoto   | Fork open-source de Terraform       |
| Pulumi        | TS, Python, Go... | Backend propio   | Lenguajes generales                 |
| Crossplane    | YAML (CRDs)       | Etcd (K8s)       | Kubernetes como plano de control    |

> Terraform y OpenTofu son practicamente identicos en sintaxis. La diferencia principal es la licencia: Terraform usa BUSL-1.1 mientras OpenTofu mantiene la licencia MPL-2.0, lo que garantiza que no pueda ser restringido por HashiCorp en el futuro.

## Herramientas Cloud-Nativas

Cada proveedor de nube ofrece su propia herramienta de IaC integrada. Estas herramientas estan optimizadas para el ecosistema del proveedor: conocen todas las funcionalidades, soportan todas las APIs y suelen ser las primeras en implementar nuevas capacidades. La contrapartida es el vendor lock-in: migrar de CloudFormation a otro orquestador requiere reescribir toda la definicion.

```yaml
# AWS CloudFormation
Resources:
  WebServer:
    Type: AWS::EC2::Instance
    Properties:
      ImageId: ami-0c55b159cbfafe1f0
      InstanceType: t3.micro
      Tags:
        - Key: Name
          Value: web-server
```

```bicep
// Azure Bicep
param location string = resourceGroup().location

resource webServer 'Microsoft.Compute/virtualMachines@2023-03-01' = {
  name: 'web-server'
  location: location
  properties: {
    hardwareProfile: {
      vmSize: 'Standard_B1s'
    }
  }
}
```

```yaml
# Google Deployment Manager
resources:
- name: web-server
  type: compute.v1.instance
  properties:
    machineType: zones/us-central1-a/machineTypes/e2-medium
    disks:
    - deviceName: boot
      type: PERSISTENT
      boot: true
      initializeParams:
        diskImage: projects/debian-cloud/global/images/debian-11
```

| Herramienta           | Proveedor | Lenguaje    | Madurez    |
|-----------------------|-----------|-------------|------------|
| CloudFormation        | AWS       | YAML/JSON   | Alta       |
| Bicep                 | Azure     | Bicep       | Alta       |
| Deployment Manager    | GCP       | YAML/JSON   | Media      |
| Oracle Resource Mgr   | Oracle    | HCL (TF)    | Media      |

> Las herramientas cloud-nativas no excluyen el uso de orquestadores externos. Un patron comun es usar Terraform para la infraestructura base (VPC, subnets) y CloudFormation para servicios especificos de AWS (Lambda, API Gateway) cuando se requiere soporte inmediato para nuevas funcionalidades.

## Herramientas de Gestion de Configuracion

Ansible, Chef y Puppet se enfocan en el estado interno de los servidores: paquetes instalados, archivos de configuracion, servicios ejecutandose y usuarios creados. No crean VMs ni redes; asumen que la infraestructura base ya existe y se encargan de configurarla y mantenerla en el estado deseado. Esta distincion es critica: confundir provisionamiento con configuracion genera arquitecturas fragiles.

```yaml
# Ansible playbook - configuracion de un servidor web
- name: Configurar servidor web
  hosts: webservers
  become: yes
  tasks:
    - name: Instalar nginx
      apt:
        name: nginx
        state: present

    - name: Copiar configuracion
      template:
        src: nginx.conf.j2
        dest: /etc/nginx/nginx.conf
      notify: reiniciar nginx

  handlers:
    - name: reiniciar nginx
      service:
        name: nginx
        state: restarted
```

```ruby
# Chef recipe - configuracion de servidor web
package 'nginx' do
  action :install
end

template '/etc/nginx/nginx.conf' do
  source 'nginx.conf.erb'
  notifies :restart, 'service[nginx]'
end

service 'nginx' do
  action [:enable, :start]
end
```

| Herramienta | Lenguaje   | Arquitectura | Enfoque                  |
|-------------|------------|--------------|--------------------------|
| Ansible     | YAML       | Agentless    | Configuracion y orquestacion |
| Chef        | Ruby DSL   | Agent (chef-client) | Configuracion declarativa  |
| Puppet      | Puppet DSL | Agent (puppet-agent) | Configuracion declarativa  |

> La frontera entre orquestadores y gestores de configuracion se ha difuminado con los anos. Terraform tiene `provisioners` que pueden ejecutar Ansible playbooks, y Ansible puede gestionar recursos cloud via colecciones. Sin embargo, la distincion conceptual sigue siendo util: orquestadores para el ciclo de vida de recursos, configuradores para el estado de los sistemas operativos.

