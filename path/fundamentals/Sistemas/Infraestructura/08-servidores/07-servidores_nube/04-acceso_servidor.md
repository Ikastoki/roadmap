# Acceso al servidor

Acceder a una instancia en la nube requiere un mecanismo de autenticacion seguro que equilibre accesibilidad con proteccion. A diferencia de los servidores fisicos tradicionales donde podian configurarse contrasenas directamente, las instancias en la nube estan disenadas para ser inaccesibles por defecto: no tienen contrasena predefinida, los puertos estan cerrados y el unico metodo de acceso inicial es a traves del mecanismo que el proveedor establece. Este diseno deliberado elimina una de las vulnerabilidades mas comunes en servidores: las contrasenas por defecto o debiles.

## SSH con claves publicas

El protocolo SSH (Secure Shell) es el mecanismo estandar de acceso remoto a servidores en la nube. La autenticacion se realiza mediante pares de claves criptograficas: una clave privada que reside exclusivamente en el equipo del administrador y una clave publica que se almacena en el servidor destino. Cuando el administrador solicita conexion, el servidor demuestra que posee la clave publica correspondiente a la clave privada del cliente, validando la identidad sin transmitir ningun secreto por la red.

La autenticacion por claves es significativamente mas segura que la autenticacion por contrasena. Las contrasenas son vulnerables a fuerza bruta, ataques de diccionario, ingenieria social y keylogging. Una clave RSA de 2048 bits o una clave ED25519 son computacionalmente inviables de forzar con la tecnologia actual. Ademas, las claves no se transmiten nunca por la red durante la autenticacion, eliminando el riesgo de captura en tránsito.

## Gestion de claves por el proveedor

Los proveedores de nube gestionan las claves SSH de forma diferente a como lo haria un administrador en un servidor fisico. En AWS, el concepto se denomina Key Pairs: al crear la instancia, el cliente selecciona un Key Pair existente o crea uno nuevo, y AWS inyecta automaticamente la clave publica en el archivo `~/.ssh/authorized_keys` del usuario por defecto de la imagen. La clave privada se descarga una sola vez al crear el Key Pair y nunca se almacena en los servidores de AWS.

Esta gestion tiene implicaciones importantes. Primero, el proveedor nunca tiene acceso a la clave privada, lo que mantiene el principio de confidencialidad: ni AWS ni el administrador pueden conectarse sin la clave privada correspondiente. Segundo, si el cliente pierde la clave privada, no hay forma de recuperarla; la unica solucion es crear un nuevo Key Pair y reconfigurar el acceso. Tercero, si una instancia se terminate y se crea una nueva, el Key Pair sigue existiendo en la cuenta pero la instancia nueva requiere que se asocie explicitamente.

> Algunos proveedores ofrecen alternativas a SSH tradicional como AWS Systems Manager Session Manager o Azure Serial Console, que permiten acceder a instancias sin abrir el puerto SSH (22) en el grupo de seguridad. Esto elimina la necesidad de exponer el servicio de acceso remoto a Internet, reduciendo significativamente la superficie de ataque.

La gestion centralizada de claves por el proveedor facilita escenarios de equipo donde multiples administradores necesitan acceso: cada uno genera su propio par de claves y lo asocia a las instancias, sin compartir claves privadas entre personas. Cuando alguien abandona el equipo, simplemente se elimina su clave publica de las instancias, revocando el acceso inmediatamente sin afectar a otros administradores.

