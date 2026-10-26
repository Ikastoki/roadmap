# Seguridad basica de contenedores

El aislamiento de contenedores no es absoluto. A diferencia de las VMs, que tienen su propio kernel separado, los contenedores comparten el kernel del host. Esto significa que una vulnerabilidad en el kernel puede potencialmente afectar a todos los contenedores. Comprender las limitaciones de seguridad y aplicar buenas practicas es fundamental para operar contenedores en produccion de forma segura.

## El problema del aislamiento compartido

```
VM:
  VM 1 (kernel Linux 5.15) ←.kernel separado→ VM 2 (kernel Linux 6.1)
  Si un contenedor compromete su kernel, no afecta al host ni a otras VMs

Contenedor:
  Contenedor 1 ─┐
  Contenedor 2 ─┤── kernel Linux 6.1 (compartido)
  Contenedor 3 ─┘
  Si un contenedor encuentra una vulnerabilidad en el kernel, puede afectar a todos
```

## Buenas practicas de seguridad

**1. Ejecutar sin privilegios de root (rootless):**

```bash
# Docker: ejecutar con usuario no root
docker run -u 1000:1000 -it ubuntu whoami
# 1000 (no root)

# Podman: ejecuta rootless por defecto
podman run -it ubuntu whoami
# root (dentro del namespace, mapeado a usuario no root en el host)
```

**2. Definir usuario no root en la imagen:**

```dockerfile
# En el Dockerfile
FROM node:20-alpine
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser
CMD ["node", "app.js"]
```

**3. Limitar recursos:**

```bash
# Evitar que un contenedor consuma todos los recursos del host
docker run -d \
  --memory 256m \
  --cpus 0.5 \
  --pids-limit 50 \
  mi-app
```

**4. No usar --privileged:**

```bash
# NUNCA en produccion sin justificacion extrema
docker run --privileged -it ubuntu bash
# Este contenedor tiene acceso al hardware del host
# Puede cargar modulos del kernel, acceder a dispositivos, etc.
```

**5. Usar imagenes de confianza:**

```bash
# Usar imagenes oficiales certificadas
docker pull postgres:16-alpine    # Imagen oficial
docker pull usuario-desconocido/app  # RIESGO: puede contener malware
```

**6. Escanear vulnerabilidades:**

```bash
# Con Docker Scout (integrado en Docker Desktop)
docker scout cves mi-app:1.0

# Con Trivy (herramienta independiente)
trivy image mi-app:1.0
```

**7. No montar el socket de Docker:**

```bash
# PELIGROSO: dar acceso al daemon de Docker desde el contenedor
docker run -v /var/run/docker.sock:/var/run/docker.sock -it docker
# Este contenedor puede crear y eliminar contenedores en el host
# Es equivalente a dar privilegios root al contenedor
```

**8. Usar perfiles de seguridad:**

```bash
# Seccomp: limitar llamadas al sistema
docker run --security-opt seccomp=profile.json mi-app

# AppArmor: limitar acciones del contenedor
docker run --security-opt apparmor=docker-default mi-app

# SELinux: etiquetado de seguridad
docker run --security-opt label=level:s0:c100,c200 mi-app
```

## Superficie de ataque

La superficie de ataque de un contenedor incluye:
- El kernel del host (compartido).
- El motor de contenedores (Docker daemon).
- Las imagenes base (vulnerabilidades conocidas).
- La configuracion de red (puertos expuestos).
- Las variables de entorno (secretos en texto plano).
- Los volumenes montados (acceso al filesystem del host).

> La seguridad de contenedores no es automatica: es una responsabilidad activa. Un contenedor por defecto tiene multiples puntos que deben ser asegurados: usuario root, sin limites de recursos, imagenes sin escanear, puertos expuestos. Aplicar buenas practicas desde el primer dia es infinitamente mas facil que corregir problemas de seguridad en produccion.

