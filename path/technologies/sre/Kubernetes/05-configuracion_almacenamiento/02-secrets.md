# Secrets

Si los ConfigMaps resuelven el problema de la configuracion no sensible, los Secrets hacen lo propio con la informacion que debe permanecer confidencial: contrasenas, tokens de API, claves SSH, certificados TLS, credenciales de bases de datos. La necesidad de gestionar estos valores de forma segura es obvia, pero en la practica la mayoria de las filtraciones de credenciales ocurren por errores tan simples como subir un archivo de configuracion a un repositorio publico o hardcodear una contrasena en una imagen de contenedor.

Un Secret en Kubernetes es un objeto similar a un ConfigMap pero con diferencias importantes: los datos se almacenan codificados en base64 (no cifrados por defecto), el acceso puede restringirse mediante RBAC, y Kubernetes puede integrarse con sistemas externos de cifrado (KMS) para proteger los datos en reposo en etcd.

## Codificacion base64 y nociones de seguridad

Es frecuente ver a desarrolladores confundir base64 con cifrado. Base64 no es mas que una codificacion de transferencia que convierte datos binarios en texto ASCII imprimible. Cualquier persona puede decodificar un valor base64 en segundos:

```bash
# Codificar
echo -n "mi-contrasena" | base64
# bWktY29udHJhc2VuYQ==

# Decodificar
echo -n "bWktY29udHJhc2VuYQ==" | base64 --decode
# mi-contrasena
```

Kubernetes codifica los valores en base64 unicamente para poder almacenar datos binarios en un campo de texto en YAML o JSON. No proporciona proteccion alguna contra lecturas no autorizadas de los objetos Secret. La seguridad real de los Secrets depende de tres capas:

1. **RBAC**: controlar que usuarios y service accounts pueden leer Secrets mediante roles de Kubernetes
2. **Cifrado en reposo de etcd**: configurar el API server para que cifre los Secrets antes de escribirlos en la base de datos
3. **Gestion externa de secretos**: usar herramientas como HashiCorp Vault, AWS Secrets Manager, Azure Key Vault o External Secrets Operator, que nunca exponen el valor real en el manifiesto

> Base64 no es seguridad. Nunca incluyas un Secret en un repositorio de Git aunque este codificado en base64. La codificacion se revierte con un comando y cualquier persona con acceso al repositorio puede obtener las credenciales.

## Tipos de Secrets

Kubernetes define varios tipos de Secret, cada uno pensado para un caso de uso especifico. El tipo se indica en el campo `type` del manifiesto y determina como Kubernetes interpreta los datos.

```yaml
# Opaque: el tipo generico para cualquier par clave-valor
apiVersion: v1
kind: Secret
metadata:
  name: generic-secret
type: Opaque
data:
  username: YWRtaW4=
  password: czNjcjN0LXAxMjM=
```

El tipo `Opaque` es el mas utilizado. Almacena pares clave-valor arbitrarios, como credenciales de base de datos, tokens de API o claves de servicios externos. Si no se especifica el tipo, Kubernetes asume `Opaque` por defecto.

```yaml
# kubernetes.io/dockerconfigjson: credenciales para registros de contenedores
apiVersion: v1
kind: Secret
metadata:
  name: registry-credentials
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: eyJhdXRocyI6...  # base64 del JSON de Docker config
```

El tipo `dockerconfigjson` almacena credenciales de autenticacion para registros de contenedores privados (Docker Hub, Amazon ECR, Google Container Registry, Azure Container Registry). Se usa en el campo `imagePullSecrets` de un Pod o se anade al service account por defecto del namespace para que todos los Pods puedan descargar imagenes del registro privado.

```bash
# Crear un Secret de tipo docker-registry de forma imperativa
kubectl create secret docker-registry regcred \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=mi-usuario \
  --docker-password=mi-contrasena \
  --docker-email=email@ejemplo.com
```

```yaml
# kubernetes.io/tls: certificados TLS
apiVersion: v1
kind: Secret
metadata:
  name: tls-certificate
type: kubernetes.io/tls
data:
  tls.crt: LS0tLS1CRUdJTi...  # Certificado en base64
  tls.key: LS0tLS1CRUdJTi...   # Clave privada en base64
```

El tipo `tls` almacena un par certificado-clave privada para terminar conexiones TLS. Se utiliza tipicamente en recursos Ingress para habilitar HTTPS. Las claves esperadas son exactamente `tls.crt` y `tls.key`.

```bash
# Crear un Secret de tipo TLS desde archivos
kubectl create secret tls mi-certificado \
  --cert=certificado.pem \
  --key=clave-privada.pem
```

## Creacion de Secrets

Al igual que los ConfigMaps, los Secrets pueden crearse de forma imperativa o declarativa. La via imperativa es util para desarrollo:

```bash
# Desde literales
kubectl create secret generic db-credentials \
  --from-literal=username=admin \
  --from-literal=password=s3cr3t-p4ss

# Desde archivos
kubectl create secret generic api-keys \
  --from-file=api-key.txt \
  --from-file=secret-key.txt

# El contenido del archivo se codifica automaticamente en base64
```

En la forma declarativa se puede usar `data` con valores ya codificados en base64, o `stringData` con valores en texto plano que Kubernetes codifica automaticamente al crear el recurso:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mi-secret
type: Opaque
stringData:
  username: admin
  password: s3cr3t-p4ss
  # stringData acepta texto plano; Kubernetes lo codifica a base64 internamente
```

Usar `stringData` en los manifiestos es mas legible y menos propenso a errores que pre-codificar manualmente los valores. Sin embargo, el manifiesto resultante sigue conteniendo las credenciales en texto plano y nunca debe subirse a un repositorio.

> La configuracion tipica de cifrado en reposo de etcd utiliza AES-CBC con una clave de 256 bits gestionada por el administrador del cluster. En proveedores cloud, la clave se gestiona a traves de KMS (AWS KMS, Azure Key Vault, GCP Cloud KMS), y el API server de Kubernetes se configura con `--encryption-provider-config` para indicar que recursos cifrar y con que proveedor.

## Montaje como archivos

Los Secrets se montan en los Pods de forma identica a los ConfigMaps. Cada clave del Secret se convierte en un archivo dentro del directorio de montaje, con el valor decodificado como contenido:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
    - name: app
      image: mi-app:1.0
      volumeMounts:
        - name: credenciales
          mountPath: /etc/credenciales
          readOnly: true
  volumes:
    - name: credenciales
      secret:
        secretName: db-credentials
```

Con este manifiesto, el contenedor encuentra los archivos `/etc/credenciales/username` y `/etc/credenciales/password`, cada uno con el valor real en texto plano. La opcion `readOnly: true` es una buena practica: evita que la aplicacion o un proceso comprometido modifique accidentalmente el contenido del Secret.

El montaje como archivo tiene la ventaja de que las actualizaciones del Secret se propagan al sistema de archivos del Pod (con el mismo retardo de sincronizacion que en los ConfigMaps). La aplicacion puede detectar cambios monitorizando los archivos.

## Montaje como variables de entorno

Tambien es posible inyectar valores del Secret como variables de entorno:

```yaml
spec:
  containers:
    - name: app
      image: mi-app:1.0
      env:
        - name: DB_USERNAME
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: username
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: password
```

No se recomienda inyectar Secrets como variables de entorno en aplicaciones que vuelcan el entorno en los logs o en paginas de error, ya que cualquier error de la aplicacion podria exponer las credenciales. El montaje como archivo es mas seguro porque el acceso requiere lectura explicita del archivo.

## ImagePullSecrets

Cuando un Pod necesita descargar una imagen de un registro privado, Kubernetes necesita autenticarse contra ese registro. El Secret de tipo `dockerconfigjson` se asocia al Pod mediante el campo `imagePullSecrets`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
    - name: app
      image: mi-registro-privado.com/mi-app:1.0
  imagePullSecrets:
    - name: registry-credentials
```

Para no tener que anadir `imagePullSecrets` a cada Pod, se puede anadir el Secret al service account `default` del namespace:

```bash
kubectl patch serviceaccount default \
  -p '{"imagePullSecrets": [{"name": "registry-credentials"}]}'
```

A partir de ese momento, todos los Pods del namespace que usen el service account por defecto heredan automaticamente la capacidad de autenticarse contra el registro privado.

## Buenas practicas

No hardcodear Secrets en los manifiestos de Kubernetes. La forma correcta de gestionar secretos en produccion depende del nivel de sofisticacion del equipo y de la infraestructura:

1. **External Secrets Operator**: sincroniza Secrets de Kubernetes desde fuentes externas (AWS Secrets Manager, Azure Key Vault, GCP Secret Manager, HashiCorp Vault). Los manifiestos solo referencian el nombre del secreto externo, no el valor.

2. **SealedSecrets**: permite cifrar un Secret de forma que solo el controlador SealedSecrets en el cluster pueda descifrarlo. El SealedSecret cifrado puede almacenarse en Git de forma segura.

3. **Helm Secrets**: extiende Helm para gestionar valores cifrados usando Mozilla SOPS con soporte para PGP, AWS KMS, GCP KMS y Azure Key Vault.

4. **Vault CSI Provider**: monta secretos de HashiCorp Vault directamente como volumentes CSI en los Pods, sin que Kubernetes almacene el valor en ningun momento.

```yaml
# Ejemplo con External Secrets Operator
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
spec:
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: db-credentials
  data:
    - secretKey: username
      remoteRef:
        key: /produccion/base-de-datos/usuario
    - secretKey: password
      remoteRef:
        key: /produccion/base-de-datos/contrasena
```

El uso de RBAC especifico para Secrets es obligatorio en entornos con multiples equipos. No todos los desarrolladores necesitan leer los Secrets de produccion. La politica minimamente recomendada es que solo los administradores del cluster y las aplicaciones que los consumen tengan permiso de lectura sobre los Secrets.

```bash
# Ver Secrets (no muestra los valores)
kubectl get secrets

# Ver los valores (codificados en base64)
kubectl get secret db-credentials -o yaml

# Ver el valor decodificado de una clave concreta
kubectl get secret db-credentials -o jsonpath="{.data.password}" | base64 --decode
```

> Rotar los Secrets periodicamente reduce el impacto de una filtracion. La rotacion puede automatizarse con herramientas como Reloader para reiniciar los Pods cuando el Secret cambia, o con Vault que permite renovar automaticamente credenciales con TTL limitado.

