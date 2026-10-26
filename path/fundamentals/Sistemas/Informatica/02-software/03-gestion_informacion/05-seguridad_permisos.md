# Seguridad y permisos

La seguridad de los archivos y directorios es un componente crítico de cualquier sistema informático. Sin mecanismos de control de acceso, cualquier usuario podría leer, modificar o eliminar cualquier archivo, incluyendo datos confidenciales, configuraciones del sistema o programas críticos. Los permisos de archivo son la primera línea de defensa contra accesos no autorizados y la corrupción accidental de datos.

La gestión de permisos varía significativamente entre sistemas operativos, pero todos comparten los mismos principios fundamentales: identificar quién solicita acceso, determinar qué tipo de acceso se solicita y verificar si ese acceso está permitido según las políticas configuradas.

## Modelos de permisos

### Unix/Linux (rwx)

El modelo de permisos Unix utiliza un sistema de tres categorías de usuario y tres tipos de permiso:

**Categorías de usuario:**
- **Propietario (*owner*):** el usuario que creó el archivo o al que se le ha asignado la propiedad.
- **Grupo (*group*):** el grupo al que pertenece el propietario. Todos los usuarios del grupo heredan los permisos de grupo.
- **Otros (*others*):** cualquier usuario que no sea el propietario ni pertenezca al grupo.

**Tipos de permiso:**
- **Lectura (*r*, read):** permite ver el contenido del archivo o listar el contenido de un directorio.
- **Escritura (*w*, write):** permite modificar el contenido del archivo o crear/eliminar archivos dentro de un directorio.
- **Ejecución (*x*, execute):** permite ejecutar el archivo (si es un programa) o acceder (entrar) a un directorio.

La representación visual de los permisos Unix utiliza una cadena de 10 caracteres:

```
-rwxr-xr--
│├─┤├─┤├─┤
│ │  │  └── Permisos de otros (lectura)
│ │  └───── Permisos de grupo (lectura y ejecución)
│ └──────── Permisos del propietario (lectura, escritura, ejecución)
└────────── Tipo de archivo (- = archivo, d = directorio, l = enlace)
```

Los permisos se modifican con el comando `chmod`:

```bash
chmod 755 archivo.sh    # rwxr-xr-x
chmod 644 documento.txt # rw-r--r--
chmod 700 script.sh     # rwx------
```

Los números representan la suma de permisos: lectura = 4, escritura = 2, ejecución = 1. Un `7` significa 4+2+1 (todos los permisos), un `5` significa 4+1 (lectura y ejecución), un `4` significa solo lectura.

### ACL en Windows

Windows utiliza listas de control de acceso (*Access Control Lists*, ACL) que permiten definir permisos mucho más granulares que el modelo Unix básico. Cada archivo y directorio tiene una ACL que lista los usuarios y grupos que pueden acceder a él, junto con los permisos específicos concedidos o denegados.

Las ACL de Windows soportan permisos como:
- **Full Control:** acceso total al archivo.
- **Modify:** lectura, escritura y ejecución, pero no cambio de permisos.
- **Read & Execute:** lectura y ejecución, pero no modificación.
- **Write:** escritura, pero no lectura del contenido existente.
- **Read:** solo lectura.

Windows también soporta herencia de permisos: los archivos y subdirectorios heredan los permisos de su directorio padre, lo que simplifica la gestión de permisos en estructuras de directorios grandes.

## Propiedad

La propiedad de un archivo determina quién tiene el control sobre sus permisos. En Unix, el propietario de un archivo puede cambiar sus permisos, transferir la propiedad a otro usuario o modificar la membresía del grupo. La propiedad se asigna automáticamente al usuario que crea el archivo.

La gestión de la propiedad es especialmente relevante en servidores donde múltiples usuarios y servicios acceden a los mismos archivos. Un error de propiedad puede impedir que un servicio web acceda a los archivos de un sitio web, o que un usuario modifique archivos que necesitan.

## Herencia de permisos

La herencia de permisos permite que los archivos y directorios nuevos hereden automáticamente los permisos de su directorio padre. En Unix, esto se logra mediante el *setgid bit* en directorios: los archivos creados dentro de un directorio con setgid heredan el grupo del directorio en lugar del grupo del usuario que los creó.

En Windows, la herencia está habilitada por defecto: los archivos creados dentro de un directorio heredan las ACL de ese directorio. La herencia puede deshabilitarse para archivos o directorios específicos que requieran permisos diferentes a los del padre.

## Integridad, respaldo y seguridad de la información

### Checksums

Un checksum es un valor calculado a partir del contenido de un archivo que se utiliza para verificar su integridad. Si el archivo se modifica (por error, corrupción o manipulación maliciosa), el checksum calculado difiere del original, detectando la alteración.

Los algoritmos de checksum comunes incluyen MD5, SHA-1, SHA-256 y SHA-3. Los hashes criptográficos (SHA-256, SHA-3) son preferibles para verificación de integridad, ya que es computacionalmente inviable encontrar dos archivos diferentes con el mismo hash.

### Backups

Las copias de seguridad (*backups*) son la última línea de defensa contra la pérdida de datos. Un backup completo protege contra fallos de hardware, ransomware, errores humanos y desastres naturales. Las estrategias de backup más comunes incluyen:

- **Backup completo:** copia todos los archivos del sistema. Es la opción más simple pero requiere más espacio y tiempo.
- **Backup incremental:** solo copia los archivos que han cambiado desde el último backup. Es más rápido y eficiente en espacio pero la restauración es más compleja.
- **Backup diferencial:** copia todos los archivos que han cambiado desde el último backup completo. Equilibra velocidad de backup y facilidad de restauración.

### Cifrado en reposo y en tránsito

**Cifrado en reposo (*at rest*):** protege los datos almacenados en disco contra acceso no autorizado. Herramientas como BitLocker (Windows), FileVault (macOS) y LUKS (Linux) cifran todo el volumen, haciendo que los datos sean ilegibles sin la clave de descifrado.

**Cifrado en tránsito (*in transit*):** protege los datos mientras se transmiten por la red. TLS/SSL cifra las comunicaciones HTTP, SSH cifra las conexiones remotas, y VPNs cifran todo el tráfico de red entre dos puntos.

