# Resolucion DNS

DNS es uno de los servicios criticos de internet: sin el, los usuarios tendrian que recordar direcciones IP en lugar de nombres de dominio. La resolucion DNS en Linux implica comprender como se configuran los servidores DNS, como se resuelven los nombres y como se diagnostican los problemas de resolucion.

En los sistemas modernos, la resolucion DNS esta gestionada por `systemd-resolved` o por el archivo `/etc/resolv.conf`. Comprender como funciona esta capa de abstraccion es esencial para resolver problemas de conectividad.

## /etc/resolv.conf

El archivo `/etc/resolv.conf` contiene la configuracion de los servidores DNS que el sistema utiliza para resolver nombres de dominio.

```
# /etc/resolv.conf
nameserver 8.8.8.8
nameserver 8.8.4.4
search ejemplo.com home.arpa
options timeout:2 attempts:3
```

- **nameserver:** la IP del servidor DNS al que se envian las consultas. Se pueden especificar multiples servidores; el sistema prueba el primero y pasa al segundo si el primero falla.
- **search:** dominios que se anaden automaticamente a las consultas no cualificadas. Si buscas `servidor` y search contiene `ejemplo.com`, se resolvera `servidor.ejemplo.com`.
- **options:** opciones como timeout y numero de reintentos.

> En Ubuntu moderno, `/etc/resolv.conf` es gestionado por `systemd-resolved` y puede ser un enlace simbolico. Editarlo directamente puede no tener efecto; es mejor configurarlo a traves de Netplan o resolvectl.

## Uso de dig

`dig` (*Domain Information Groper*) es la herramienta mas completa para consultar y diagnosticar DNS.

```bash
# Consulta basica A
dig www.google.com

# Consulta MX (servidores de correo)
dig MX gmail.com

# Consulta CNAME (alias)
dig CNAME www.github.com

# Consulta inversa (IP a nombre)
dig -x 8.8.8.8

# Usar un servidor DNS especifico
dig @8.8.8.8 www.google.com

# Salida concisa (solo la respuesta)
dig +short google.com

# Ver tiempos de respuesta
dig +stats google.com

# Consulta TCP (en lugar de UDP)
dig +tcp google.com
```

La salida de `dig` incluye:

- **ANSWER SECTION:** la respuesta a la consulta.
- **AUTHORITY SECTION:** los servidores autoritativos para el dominio.
- **ADDITIONAL SECTION:** informacion adicional (IPs de los nameservers).
- **Query time:** tiempo de respuesta en milisegundos.
- **SERVER:** el servidor DNS que respondio.

## Uso de nslookup

`nslookup` es una alternativa mas simple a `dig`, util para consultas rapidas.

```bash
# Consulta basica
nslookup google.com

# Consulta MX
nslookup -type=MX gmail.com

# Usar un servidor DNS especifico
nslookup google.com 8.8.8.8

# Consulta inversa
nslookup 8.8.8.8
```

## systemd-resolved

En Ubuntu 16.04+ y otras distribuciones modernas, `systemd-resolved` gestiona la resolucion DNS como un servicio del sistema.

```bash
# Ver estado del resolved
resolvectl status

# Ver la resolucion de un nombre
resolvectl query google.com

# Ver las transacciones de cache
resolvectl statistics

# Flushear la cache DNS
resolvectl flush-caches

# Ver servidores DNS configurados
resolvectl dns
```

systemd-resolved mantiene una cache local de consultas DNS recientes, reduciendo el numero de consultas a servidores externos y mejorando la latencia de resolucion.

## Problemas comunes de DNS

**DNS no resuelve:** verificar `/etc/resolv.conf`, probar con `dig @8.8.8.8`, comprobar conectividad basica con `ping 8.8.8.8`.

**DNS lento:** verificar la latencia al servidor DNS con `dig +stats`, cambiar a un servidor DNS mas rapido (Cloudflare 1.1.1.1, Google 8.8.8.8), verificar la cache de systemd-resolved.

**Resolucion inconsistente:** puede deberse a DNS caching con TTL bajo, multiples servidores DNS con respuestas diferentes, o propagacion de cambios DNS recientes.

**Error SERVFAIL:** el servidor DNS no puede resolver la consulta. Puede ser un problema del servidor DNS autoritativo o un bloqueo de firewall hacia el puerto 53 UDP/TCP.

> DNS es el servicio silencioso que sostiene internet: cuando funciona, nadie lo nota; cuando falla, todo se detiene. Comprender la resolucion DNS, las herramientas de diagnostico y la configuracion del sistema es indispensable para cualquier administrador de sistemas.

