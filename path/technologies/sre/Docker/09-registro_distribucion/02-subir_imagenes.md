# Subir imagenes a un registro

Construir una imagen localmente es solo el primer paso del ciclo de vida de un contenedor. Para que esa imagen este disponible en otros entornos --el servidor de staging, el cluster de Kubernetes o el equipo de otro desarrollador-- es necesario subirla (push) a un registro. El proceso implica tres operaciones consecutivas: etiquetar la imagen con la referencia adecuada, autenticarse contra el registro destino y transferir las capas que componen la imagen.

## Etiquetado de imagenes con docker tag

El comando `docker tag` asigna un nuevo nombre y etiqueta a una imagen existente sin modificar sus capas. Una imagen puede tener multiples etiquetas que apuntan al mismo identificador interno. Esto permite que una misma imagen se publique en distintos repositorios o con distintas versiones sin duplicar datos.

```bash
# Crear una etiqueta para una imagen local
docker tag mi-app:latest usuario/mi-app:1.0.0

# Verificar que ambas referencias comparten el mismo IMAGE ID
docker images --digests
```

La sintaxis completa de `docker tag` es:

```
docker tag <imagen-origen> <referencia-destino>
```

Donde `<referencia-destino>` sigue el formato `[registro/][usuario/]repositorio:tag`. El registro se omite cuando el destino es Docker Hub. Para cualquier otro registro, debe incluirse la URL completa:

```bash
# Etiquetar para GitHub Container Registry
docker tag mi-app:latest ghcr.io/mi-usuario/mi-app:1.0.0

# Etiquetar para un registry privado
docker tag mi-app:latest registry.miempresa.com/mi-app:1.0.0
```

> Etiquetar no es lo mismo que copiar. Todos los alias de una misma imagen comparten el mismo hash SHA256 de manifiesto y las mismas capas. Ocupar el mismo espacio en disco hasta que se suben a registros diferentes.

## Autenticacion con docker login

Antes de poder subir una imagen a un registro, el cliente Docker debe demostrar que tiene permisos de escritura. El comando `docker login` inicia una sesion autenticada y almacena las credenciales en `~/.docker/config.json`.

```bash
# Login interactivo en Docker Hub
docker login

# Login con usuario y contrasena (no interactivo, util en scripts)
docker login -u usuario -p "$TOKEN"

# Login en un registro privado
docker login registry.miempresa.com -u usuario -p "$TOKEN"
```

En entornos de CI/CD, el metodo recomendado es utilizar tokens de acceso personal en lugar de contrasenas. Docker Hub permite generar tokens desde el panel de usuario, y estos tokens pueden revocarse individualmente sin afectar a la cuenta principal. En GitHub Container Registry, el token es un Personal Access Token (PAT) con el scope `write:packages`.

El fichero `~/.docker/config.json` almacena las credenciales codificadas en base64. Docker permite utilizar ayudantes de credenciales (credential helpers) que delegan el almacenamiento a sistemas mas seguros como el llavero del sistema operativo (macOS Keychain, Linux secret-tool, Windows Credential Manager).

```bash
# Configurar un credential helper para evitar almacenar tokens en texto plano
# En ~/.docker/config.json
{
  "credHelpers": {
    "https://index.docker.io/v1/": "osxkeychain"
  }
}
```

## Subida de imagenes con docker push

Una vez autenticado y con la imagen correctamente etiquetada, el comando `docker push` transfiere la imagen al registro. El proceso consta de varias fases:

```bash
docker push usuario/mi-app:1.0.0
```

En primer lugar, el cliente Docker envia el manifiesto de la imagen al registro, que incluye los metadatos (configuracion, variables de entorno, comando de entrada) y la lista de capas con sus respectivos digests. A continuacion, el registro comprueba que capas ya estan almacenadas y cuales deben transferirse. Solo las capas nuevas se envian realmente; las que ya existen en el registro (porque otra imagen las comparte) se omiten, lo que acelera la subida y ahorra ancho de banda.

Cada capa se comprime en formato tar.gz antes de enviarse y se verifica mediante su digest SHA256. Si la transferencia se interrumpe, Docker puede reanudarla desde el punto en que se detuvo gracias a la naturaleza incremental del protocolo.

```bash
# Subir todas las etiquetas de un repositorio
docker push --all-tags usuario/mi-app
```

## Push multiplataforma

Cuando una imagen se construye para multiples arquitecturas (amd64, arm64, armv7) mediante `docker buildx`, el push se realiza con un manifiesto multiple (multi-architecture manifest). Este manifiesto enumera las variantes disponibles y permite que `docker pull` seleccione automaticamente la version adecuada para la arquitectura del cliente.

```bash
# Construir y subir imagenes multiplataforma en un solo comando
docker buildx build --platform linux/amd64,linux/arm64 \
  -t usuario/mi-app:1.0.0 \
  --push .
```

El flag `--push` combina la construccion y la subida, evitando el paso intermedio de `docker push`. El resultado es un unico tag que, al descargarse, proporciona la capa correspondiente a la arquitectura del sistema que realiza el pull.

## Buenas practicas en la subida de imagenes

El proceso de push no debe ser un paso manual en entornos profesionales. La subida debe automatizarse dentro de pipelines de CI/CD, pero incluso en esos pipelines conviene seguir ciertas normas. La imagen debe etiquetarse con un identificador unico, tipicamente el hash del commit de Git o un numero de version semantica, antes de subirse. La etiqueta `latest` puede actualizarse como un alias adicional, pero nunca debe ser la unica referencia.

```bash
# Etiquetado con version semantica y commit hash
docker tag mi-app:build-1234 usuario/mi-app:1.0.0
docker tag mi-app:build-1234 usuario/mi-app:latest
docker push usuario/mi-app:1.0.0
docker push usuario/mi-app:latest
```

Otra practica recomendada es escanear la imagen antes de subirla para detectar vulnerabilidades conocidas. Herramientas como Trivy o Docker Scout pueden integrarse en el pipeline para rechazar el push si la imagen supera un umbral de criticidad.

