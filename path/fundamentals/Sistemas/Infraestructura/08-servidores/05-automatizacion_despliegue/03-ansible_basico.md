# Ansible Basico

Ansible se convierte en la herramienta de referencia para gestion de configuracion porque resuelve un problema que las demas no abordan con la misma simplicidad: como configurar servidores remotos sin instalar nada en ellos. Mientras Puppet y Chef requieren un agente instalado y configurado en cada nodo, Ansible se conecta por SSH, ejecuta modulos Python transitorios en el servidor remoto y recupera el resultado. Esta arquitectura agentless elimina la necesidad de mantenimiento de agentes, certificados de seguridad adicionales y procesos daemon en cada servidor. El punto de partida es el inventario: una lista de los servidores que Ansible debe gestionar, junto con los playbooks que describen que debe hacer en cada uno.

## El archivo de inventario

El inventario es el documento maestro de infraestructura. Define que servidores existen, a cuales se puede acceder y como se agrupan. Sin inventario, Ansible no tiene a donde conectarse. El formato mas simple es un archivo de texto plano donde cada linea es un hostname o una direccion IP.

```ini
[webservers]
web01.example.com
web02.example.com

[dbservers]
db01.example.com
db02.example.com
```

Las secciones entre corchetes definen grupos. Los grupos permiten ejecutar playbooks contra conjuntos especificos de servidores: todos los webservers, todos los dbservers, o todos los servidores a la vez. Esta agrupacion es fundamental en entornos de produccion donde diferentes servidores tienen roles distintos.

### Inventario dinamico

En entornos cloud donde los servidores se crean y destruyen frecuentemente, mantener un archivo de inventario estatico es inviable. Ansible soporta **inventarios dinamicos**: scripts o plugins que consultan APIs (AWS EC2, Azure, GCP, VMware) para generar la lista de servidores en tiempo de ejecucion. Esto permite que un playbook se ejecute contra todos los servidores que existen en un momento dado, sin importar cuales se han creado o eliminado recientemente.

```ini
[webservers]
web01 ansible_host=192.168.1.10 ansible_user=deploy ansible_ssh_private_key_file=~/.ssh/id_rsa
```

Las variables `ansible_host`, `ansible_user` y `ansible_ssh_private_key_file` personalizan la conexion para cada host. Esto es util cuando los servidores tienen direcciones IP distintas, usuarios diferentes o llaves SSH especificas.

## Playbooks: el corazon de Ansible

Un playbook es un archivo YAML que define un conjunto de tareas para ejecutar en hosts especificos. Cada playbook tiene una estructura jerarquica: un listado de hosts, variables, y una secuencia de tareas. Cada tarea invoca un modulo de Ansible que realiza una accion concreta.

```yaml
---
- name: Configurar servidores web
  hosts: webservers
  become: yes
  vars:
    paquete_principal: nginx

  tasks:
    - name: Instalar nginx
      ansible.builtin.apt:
        name: "{{ paquete_principal }}"
        state: present
        update_cache: yes

    - name: Copiar configuracion de nginx
      ansible.builtin.copy:
        src: files/nginx.conf
        dest: /etc/nginx/nginx.conf
        owner: root
        group: root
        mode: '0644'
      notify: Reiniciar nginx

    - name: Asegurar que nginx este iniciado
      ansible.builtin.service:
        name: nginx
        state: started
        enabled: yes

  handlers:
    - name: Reiniciar nginx
      ansible.builtin.service:
        name: nginx
        state: restarted
```

Cada bloque `- name` en el listado de `tasks` es una tarea individual. Ansible la ejecuta en orden, y si alguna falla, detiene la ejecucion en ese host (a menos que se configure `ignore_errors`). El modulo `apt` gestiona paquetes en Debian/Ubuntu, `copy` transfiere archivos desde la maquina de control al servidor remoto, y `service` gestiona el estado de servicios systemd.

### Handlers: reacciones a cambios

Los handlers se ejecutan solo cuando una tarea notifica un cambio. En el ejemplo anterior, si el archivo `nginx.conf` no cambia, la tarea de copia no notifica nada y el handler no se ejecuta. Si el archivo cambia, la tarea notifica al handler `Reiniciar nginx`, y este se ejecuta una vez al final del playbook, no despues de cada cambio. Este mecanismo evita reinicios innecesarios.

### Variables y templates

Las variables permiten parametrizar playbooks para que funcionen en distintos entornos. En lugar de escribir rutas o configuraciones fijas, se definen variables que se sustituyen durante la ejecucion. Para archivos de configuracion complejos, Ansible utiliza **Jinja2 templates**: archivos con placeholders que se rellenan con valores de variables.

```yaml
- name: Configurar virtual host
  ansible.builtin.template:
    src: templates/vhost.conf.j2
    dest: "/etc/nginx/sites-available/{{ dominio }}.conf"
    owner: root
    group: root
    mode: '0644'
```

El archivo `vhost.conf.j2` puede contener expresiones Jinja2 como `{{ dominio }}` o `{% for sitio in sitios %}` que se resuelven con las variables del playbook.

## El modelo SSH de ejecucion

Cuando Ansible ejecuta un playbook, el proceso es el siguiente:

1. Ansible lee el inventario y selecciona los hosts objetivo.
2. Para cada host, establece una conexion SSH.
3. Copia un modulo Python al servidor remoto a traves de SFTP o SCP.
4. Ejecuta el modulo Python en el servidor remoto.
5. Recibe el resultado (JSON) y elimina el modulo.
6. Repite el proceso para la siguiente tarea.

Este ciclo se repite para cada tarea en cada host. La eliminacion automatica de modulos garantiza que no queda nada persistente en el servidor remoto despues de la ejecucion. El unico requisito es que el servidor tenga Python instalado (casi todas las distribuciones Linux lo incluyen por defecto) y SSH accesible desde la maquina de control.

> La ejecucion por defecto es secuencial entre hosts. El parametro `forks` (por defecto 5) controla cuantos hosts se procesan en paralelo. En entornos grandes, aumentar `forks` reduce significativamente el tiempo de ejecucion.

## Modulos esenciales

Ansible incluye cientos de modulos, pero un conjunto reducido cubre la mayoria de tareas de administracion:

- **apt / yum**: gestion de paquetes en sistemas Debian y Red Hat.
- **copy / template**: transferencia de archivos y plantillas.
- **service / systemd**: control de servicios.
- **user / group**: gestion de usuarios y grupos.
- **file**: creacion de directorios, archivos y enlaces con permisos especificos.
- **lineinfile**: insercion, eliminacion o modificacion de lineas en archivos de texto.
- **command / shell**: ejecucion de comandos arbitrarios (cuando no hay modulo especifico).

> El modulo `command` ejecuta comandos sin shell, lo cual es mas seguro pero no soporta redirecciones ni tuberias. El modulo `shell` si lo hace, pero debe usarse con precaucion para evitar inyeccion de comandos.

## Ejecucion y dry-run

Ansible permite ejecutar playbooks con el flag `--check` para simular los cambios sin aplicarlos. Esto es util para verificar que el playbook hara lo esperado antes de ejecutarlo en produccion. Combinado con `--diff`, muestra exactamente que archivos cambiarian y que lineas se modificarian.

```bash
ansible-playbook -i inventory.ini playbook.yml --check --diff
```

La combinacion de dry-run y diff es la forma mas segura de validar playbooks antes de un despliegue real. Sin esta practica, un error en una variable o un template puede causar configuraciones incorrectas en multiples servidores simultaneamente.

