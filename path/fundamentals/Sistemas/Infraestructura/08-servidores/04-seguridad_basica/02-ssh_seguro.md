# SSH seguro

SSH (Secure Shell) es el protocolo de acceso remoto estandar en servidores Linux. Casi todos los servidores lo ejecutan en el puerto 22, lo que lo convierte en el objetivo principal de bots y atacantes que realizan fuerza bruta masiva contra ese puerto las 24 horas del dia. Un servidor SSH con configuracion por defecto —contraseña como metodo de autenticacion, root habilitado, puerto estandar— es una invitacion abierta al compromiso.

La seguridad de SSH no depende de una sola medida, sino de varias capas que se apilan: cambiar el puerto reduce el trafico automatizado, deshabilitar la autenticacion por contraseña elimina la fuerza bruta, y restringir el acceso de root limita el dano potencial si las credenciales de un usuario son comprometidas.

## Autenticacion por clave SSH

La autenticacion por clave reemplaza las contraseñas con un par de claves criptograficas asimetricas. El servidor almacena la clave publica del usuario en `~/.ssh/authorized_keys`, mientras que la clave privada permanece en el equipo del cliente. Cuando el cliente se conecta, el servidor envia un desafio criptografico que solo la clave privada puede resolver. Esto implica varias ventajas sobre las contraseñas: la clave privada nunca se transmite por la red (no hay riesgo de sniffing), la longitud de las claves RSA o Ed25519 hace que un ataque por fuerza bruta sea computamentalmente inviable, y cada usuario puede tener multiples claves para diferentes equipos.

Para generar un par de claves:

```bash
ssh-keygen -t ed25519 -C "usuario@servidor"
```

La clave publica se copia al servidor con `ssh-copy-id` o manualmente añadiendola a `authorized_keys`. Una vez verificado que la conexion por clave funciona correctamente, se procede a deshabilitar las contraseñas en el servidor.

## Cambio de puerto predeterminado

El puerto 22 es el objetivo predilecto de escaneos masivos e intentos de fuerza bruta automatizados. Cambiarlo a un puerto no estandar (por ejemplo, 2222 o 42022) elimina inmediatamente la mayor parte de ese trafico basura. No es una medida de seguridad fuerte por si sola —cualquier escaneo completo de puertos lo descubrira—, pero reduce significativamente el ruido en los logs y el consumo de recursos del sistema.

El cambio se realiza en `/etc/ssh/sshd_config` modificando la directiva `Port`. Tras reiniciar el servicio SSH, el nuevo puerto debe indicarse explicitamente en las conexiones del cliente o configurarse en `~/.ssh/config`.

> Cambiar el puerto sin antes verificar que el nuevo esta abierto en el firewall puede bloquear el acceso remoto al servidor. Se recomienda mantener una sesion SSH activa mientras se realiza el cambio para poder revertirlo si algo falla.

## Deshabilitar login de root y autenticacion por contraseña

Permitir el login directo de root por SSH es una practica arriesgada: si un atacante compromete la clave de root, tiene control total del servidor sin necesidad de escalar privilegios. La recomendacion es deshabilitar el root login y forzar que los usuarios accedan con su propia cuenta, elevando privilegios despues con `sudo` cuando sea necesario.

La deshabilitacion de la autenticacion por contraseña se configura con las directivas en `/etc/ssh/sshd_config`:

```
PermitRootLogin no
PasswordAuthentication no
ChallengeResponseAuthentication no
UsePAM no
```

Despues de aplicar estos cambios, solo sera posible conectarse mediante claves SSH. Esto requiere que cada usuario tenga su par de claves configurado antes de cerrar la sesion de administrador actual, ya que de lo contrario quedaria bloqueado fuera del servidor.

El flujo de una sesion SSH con autenticacion por clave es el siguiente: el cliente envia su identidad de usuario y el nombre de la clave publica que quiere usar; el servidor busca esa clave en `authorized_keys` y envia un nonce cifrado con la clave publica; el cliente descifra el nonce con su clave privada y lo devuelve al servidor, que verifica la coincidencia. Todo esto ocurre en el handshake TLS/SSH y garantiza que nadie pueda suplantar al usuario sin poseer la clave privada.

