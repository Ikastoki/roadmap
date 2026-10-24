# SSH: conexion remota segura

SSH (Secure Shell) es el estandar de facto para la administracion remota de sistemas Linux. Proporciona un canal cifrado para ejecutar comandos, transferir archivos y tunelizar trafico a traves de redes inseguras. Sin SSH, la administracion de servidores en produccion seria practicamente imposible: no hay consola fisica para cada maquina, y cualquier protocolo sin cifrado (como telnet) expone credenciales y datos en texto plano.

## Conexion basica

La sintaxis fundamental de SSH establece una sesion interactiva con un usuario y un host remotos:

```bash
ssh usuario@servidor         # Conexion usando el puerto por defecto (22)
ssh -p 2222 user@host        # Puerto personalizado
ssh -v user@host             # Modo verbose (depuracion)
ssh -vvv user@host           # Verbosidad maxima (trazas de autenticacion)
```

La primera vez que se conecta a un servidor, SSH muestra la huella digital (fingerprint) de la clave del host remoto y pide confirmacion. Esta verificacion protege contra ataques man-in-the-middle. La clave aceptada se almacena en `~/.ssh/known_hosts` para conexiones futuras.

SSH soporta multiples metodos de autenticacion que se negocian en el handshake: contrasena, claves publicas, GSSAPI (Kerberos), o autenticacion con tarjeta inteligente. El servidor acepta el primer metodo que tenga exito, en el orden definido en `/etc/ssh/sshd_config`.

## Autenticacion por claves SSH

La autenticacion por contrasena tiene desventajas: las contrasenas debiles son vulnerables a fuerza bruta, y escribir contrasenas en cada conexion es tedioso cuando se gestionan multiples servidores. La autenticacion por clave publica resuelve ambos problemas.

El proceso comienza generando un par de claves en el cliente:

```bash
ssh-keygen -t ed25519                   # Clave Ed25519 (recomendada)
ssh-keygen -t rsa -b 4096               # Clave RSA de 4096 bits
ssh-keygen -t ed25519 -f ~/.ssh/miclave -C "comentario"  # Personalizada
```

La clave privada (`id_ed25519`) nunca debe compartirse. La clave publica (`id_ed25519.pub`) se copia al servidor remoto:

```bash
ssh-copy-id user@servidor               # Copia la clave publica al servidor
ssh-copy-id -i ~/.ssh/miclave.pub user@servidor  # Clave especifica
```

Este comando anade la clave publica al archivo `~/.ssh/authorized_keys` del usuario remoto. A partir de ese momento, el cliente puede autenticarse sin contrasena: SSH usa la clave privada para firmar un desafio criptografico que el servidor verifica con la clave publica almacenada.

> La seguridad de la clave privada se refuerza con una frase de contrasena (`passphrase`). Para evitar tener que escribirla en cada conexion, `ssh-agent` la mantiene en memoria durante la sesion. `ssh-add ~/.ssh/id_ed25519` la carga en el agente.

## Configuracion del servidor SSH

El servidor SSH se configura en `/etc/ssh/sshd_config`. Las directivas mas relevantes para la seguridad son:

```
Port 2222                      # Cambia el puerto por defecto (22)
PermitRootLogin no             # Deshabilita login directo como root
PasswordAuthentication no      # Deshabilita autenticacion por contrasena
PubkeyAuthentication yes       # Habilita solo claves publicas
AllowUsers alice bob           # Restringe usuarios que pueden conectar
MaxAuthTries 3                 # Limita intentos de autenticacion
ClientAliveInterval 300        # Timeout de inactividad (segundos)
```

Despues de modificar la configuracion, se aplica con `sudo systemctl restart sshd` y se verifica que no haya errores de sintaxis con `sshd -t`.

## Conexiones avanzadas y tunelizacion

SSH permite mucho mas que una terminal remota. Algunas operaciones avanzadas:

```bash
# Ejecutar un comando remoto sin sesion interactiva
ssh user@servidor "df -h | grep /dev/sda"

# Copiar archivos con scp (heredero de rcp, sobre SSH)
scp archivo.txt user@servidor:/ruta/destino/
scp -r directorio/ user@servidor:/ruta/destino/

# Copiar archivos con rsync sobre SSH (recomendado para grandes volumenes)
rsync -avz -e ssh origen/ user@servidor:/destino/

# Forwarding de puerto local (tunel SSH)
ssh -L 8080:localhost:80 user@servidor  # Accede a :80 del servidor via 8080 local

# Forwarding de puerto remoto
ssh -R 3000:localhost:3000 user@servidor  # Expone puerto local en el servidor

# Proxy SOCKS dinamico (navegacion segura a traves del servidor)
ssh -D 1080 user@servidor

# Forwarding de agente SSH (usar claves locales en conexiones desde el servidor)
ssh -A user@servidor
```

Los tuneles SSH son especialmente utiles para acceder a servicios internos detras de un firewall o para cifrar trafico de aplicaciones que no soportan cifrado nativo.

> Herramientas como `rsync` sobre SSH son el estandar para backups remotos y sincronizacion de directorios. La combinacion de `--link-dest` permite backups incrementales que ocupan el mismo espacio que copias completas.

