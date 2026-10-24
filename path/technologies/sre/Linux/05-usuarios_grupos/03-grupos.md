# Administracion de grupos

Los grupos permiten agrupar usuarios para compartir permisos sobre archivos y recursos. En lugar de asignar permisos a cada usuario individualmente, se asigna un grupo y se anaden los usuarios a ese grupo. Esto simplifica la administracion y sigue el principio de menor privilegio: los permisos se gestionan a nivel de grupo, no de usuario.

Cada usuario tiene exactamente un grupo primario (definido en `/etc/passwd`) y puede pertenecer a cero o mas grupos secundarios (definidos en `/etc/group`). Cuando un usuario crea un archivo, este hereda el grupo primario del usuario como grupo propietario.

## groupadd

`groupadd` crea un nuevo grupo en el sistema. Al igual que `useradd`, actualiza `/etc/group` y asigna un GID automaticamente.

```bash
# Crear un grupo con el siguiente GID disponible
sudo groupadd desarrolladores

# Crear un grupo con GID especifico
sudo groupadd -g 1050 devops

# Crear un grupo del sistema (GID por debajo de SYS_GID_MAX en login.defs)
sudo groupadd -r serviciogrupo
```

## groupmod y groupdel

`groupmod` modifica los atributos de un grupo existente y `groupdel` lo elimina.

```bash
# Cambiar el nombre de un grupo
sudo groupmod -n devops_nuevo devops

# Cambiar el GID de un grupo
sudo groupmod -g 1100 desarrolladores

# Eliminar un grupo (no debe tener usuarios con este grupo como primario)
sudo groupdel desarrolladores
```

> No se puede eliminar un grupo si es el grupo primario de algun usuario. Primero hay que reasignar el grupo primario de esos usuarios con `usermod -g`.

## Grupos primarios y secundarios

El grupo primario se asigna al crear el usuario con `useradd -g` o se modifica con `usermod -g`. Los grupos secundarios se gestionan con `usermod -G` o `gpasswd`.

```bash
# Crear usuario con grupo primario especifico
sudo useradd -m -g desarrolladores -s /bin/bash juan

# Cambiar el grupo primario de un usuario existente
sudo usermod -g desarrolladores maria

# Anadir grupos secundarios (conservando los existentes)
sudo usermod -aG docker,sudo juan

# Ver los grupos de un usuario
groups juan
id juan
```

La diferencia entre grupo primario y secundario es sutil pero importante:

- El grupo primario determina el grupo propietario de los archivos que crea el usuario.
- Los grupos secundarios determinan permisos adicionales: un usuario puede acceder a un archivo si pertenece al grupo propietario del archivo, ya sea como grupo primario o secundario.
- Al crear un usuario con `useradd` sin especificar `-g`, se crea automaticamente un grupo con el mismo nombre que el usuario y se asigna como su grupo primario (comportamiento de User Private Groups).

## gpasswd

`gpasswd` administra los miembros y la contrasena de un grupo. Aunque el concepto de contrasena de grupo existe, rara vez se usa en la practica.

```bash
# Anadir un usuario a un grupo
sudo gpasswd -a juan docker

# Remover un usuario de un grupo
sudo gpasswd -d juan docker

# Establecer una contrasena para el grupo (poco comun)
sudo gpasswd desarrolladores

# Definir administradores del grupo
sudo gpasswd -A juan desarrolladores
```

> `gpasswd -a` y `usermod -aG` son equivalentes funcionales. La diferencia es que `gpasswd` actua sobre un unico grupo cada vez, mientras que `usermod` puede anadir a varios grupos simultaneamente.

## Ver miembros de un grupo

```bash
# Ver miembros de un grupo desde /etc/group
grep "^docker:" /etc/group

# Listar todos los grupos y sus miembros
cat /etc/group

# Ver los grupos de un usuario especifico
groups juan
id juan
```

El comando `id` muestra tanto el UID como los GIDs del usuario, distinguiendo entre primario y secundarios:

```
uid=1001(juan) gid=1001(juan) groups=1001(juan),27(sudo),999(docker)
```

El primer grupo despues de `gid=` es el grupo primario. Los grupos listados en `groups=` incluyen tanto el primario como los secundarios.

