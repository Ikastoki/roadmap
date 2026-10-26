# Gestion de secretos en orquestadores

Mientras los ConfigMaps almacenan configuracion no sensible, los Secrets constituyen el mecanismo dedicado para gestionar datos criticos como contrasenas de bases de datos, tokens de autenticacion, certificados TLS y claves de cifrado. En Kubernetes, un Secret es un recurso de la API que almacena datos codificados en base64, y que se puede inyectar en los pods de manera similar a los ConfigMaps. Sin embargo, es fundamental entender que la codificacion base64 no es cifrado: cualquier persona con acceso al cluster puede decodificar los valores almacenados en un Secret. Por esta razon, las mejores practicas de seguridad exigen complementar los Secrets nativos con mecanismos de cifrado en reposo en etcd y, en entornos de produccion exigentes, integrar soluciones externas de gestion de secretos como HashiCorp Vault o los servicios nativos de los proveedores cloud.

## Estructura de un Secret

Un Secret en Kubernetes sigue la misma estructura basada de API que otros recursos, pero almacena sus valores codificados en base64. Es importante notar que los valores en la manifestacion YAML deben estar codificados manualmente antes de ser insertados, o bien la herramienta `kubectl` puede codificarlos automaticamente al crear el recurso desde un fichero o literales. A diferencia de los ConfigMaps, los Secrets tienen restricciones de acceso por defecto: solo los pods en el mismo namespace pueden acceder a ellos, y el kubelet solo los transmite a traves de canales seguros cuando es estrictamente necesario.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
  namespace: produccion
  labels:
    app: mi-aplicacion
type: Opaque  # Tipo estandar para datos arbitrarios
data:
  # Valores codificados en base64
  # echo -n 'mi_contrasena' | base64
  DATABASE_PASSWORD: bWlfbm9tYnJlX2NvbnRyYXNlbmE=
  REDIS_PASSWORD: ZWRpc19yZWRpc19jb250cmFzZW5h
  API_SECRET_KEY: c2VjdXJlX3JhbmRvbV9rZXlfMzIgYml0cw==
  
  # Certificado TLS (el tipo especial manejara esto)
  # tls.crt: <certificado codificado en base64>
  # tls.key: <clave privada codificada en base64>

# Tambien se puede definir con stringData (no requiere codificacion)
stringData:
  DATABASE_URL: "postgresql://admin:bWlfbm9tYnJl@postgresql.cluster.local:5432/app_prod"
  SPRING_CONFIG_IMPORT: "vault://secret/data/app/config"
```

> Los valores en `stringData` son de solo escritura: nunca se muestran cuando se consulta el Secret existente. Solo `data` contiene los valores reales tras la creacion del recurso.

## Inyeccion de secrets en pods

La inyeccion de secrets en pods sigue los mismos patrones que los ConfigMaps, con la adicion de que los Secrets pueden montarse como volumenes con permisos de fichero restringidos, lo que es especialmente util para certificados TLS que deben ser legibles por procesos que ejecutan con usuarios no-root.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-con-secrets
  namespace: produccion
spec:
  serviceAccountName: app-service-account
  containers:
    - name: aplicacion
      image: mi-app:2.1.0
      env:
        # Secret como variable de entorno individual
        - name: DATABASE_PASSWORD
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: DATABASE_PASSWORD
        - name: REDIS_PASSWORD
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: REDIS_PASSWORD
              optional: true
      
      volumeMounts:
        # Montaje de certificado TLS con permisos especificos
        - name: tls-volume
          mountPath: /etc/tls
          readOnly: true
        # Montaje de secretos generales como ficheros
        - name: secrets-volume
          mountPath: /etc/secrets
          readOnly: true

  volumes:
    - name: tls-volume
      secret:
        secretName: app-tls-cert
        defaultMode: 0400  # Solo lectura para el propietario
        items:
          - key: tls.crt
            path: server.crt
          - key: tls.key
            path: server.key
    - name: secrets-volume
      secret:
        secretName: app-secrets
        defaultMode: 0440
```

> El campo `serviceAccountName` es relevante cuando se usa RBAC para restringir que pods especificos puedan acceder a secrets determinados. Sin una politica de control de acceso, cualquier pod en el mismo namespace puede leer cualquier secret.

## Cifrado en reposo de etcd

El almacenamiento por defecto de Secrets en etcd esta codificado en base64, pero no cifrado criptograficamente. Si un atacante obtiene acceso directo a etcd (por ejemplo, a traves de un backup o acceso al nodo que ejecuta etcd), puede decodificar todos los secrets. Para mitigar esto, Kubernetes soporta cifrado en reposo para etcd, que cifra los objetos Secret antes de escribirlos en el almacenes de datos.

```yaml
# EncryptionConfiguration para etcd
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
    providers:
      # Cifrado con AES-CBC (recomendado: AES-GCM)
      - aescbc:
          keys:
            - name: clave-primaria
              secret: <base64-de-clave-de-32-bytes>
              rotation: true
            - name: clave-secundaria
              secret: <base64-de-clave-anterior>
      # Fallback: Secrets sin cifrar (para migracion)
      - identity: {}
```

```bash
# Generar una clave de cifrado aleatoria
head -c 32 /dev/urandom | base64

# Aplicar el cifrado a todos los secrets existentes
kubectl get secrets --all-namespaces -o json | \
  kubectl replace -f - --raw /api/v1/namespaces/{namespace}/secrets/{name}
```

## Integracion con HashiCorp Vault

Para entornos de produccion donde los requisitos de seguridad exigen gestores de secretos externos con control de acceso granular, rotacion automatica y auditoria centralizada, HashiCorp Vault representa la solucion mas adoptada. Vault opera como un servicio independiente que almacena y gestiona secretos con politicas de acceso basadas en identidad, y puede integrarse con Kubernetes mediante el metodo de autenticacion nativo, que valida la identidad del service account del pod contra Vault.

```yaml
# Politica de Vault para la aplicacion
path "secret/data/app/production/*" {
  capabilities = ["read", "list"]
}

path "database/creds/app-role" {
  capabilities = ["read"]
}
```

```yaml
# Pod con inyeccion de secrets desde Vault
apiVersion: v1
kind: Pod
metadata:
  name: app-with-vault
  namespace: produccion
  annotations:
    # Vault Agent Sidecar inyectara secrets automaticamente
    vault.hashicorp.com/agent-inject: "true"
    vault.hashicorp.com/role: "app-production-role"
    vault.hashicorp.com/agent-inject-secret-db-creds: "database/creds/app-role"
    vault.hashicorp.com/agent-inject-template-db-creds: |
      {{- with secret "database/creds/app-role" -}}
      export DB_USERNAME="{{ .Data.username }}"
      export DB_PASSWORD="{{ .Data.password }}"
      {{- end -}}
    vault.hashicorp.com/agent-inject-secret-config: "secret/data/app/production/config"
    vault.hashicorp.com/agent-inject-permissions-config: "0400"
    vault.hashicorp.com/agent-pre-populate-only: "false"
spec:
  serviceAccountName: vault-auth
  containers:
    - name: aplicacion
      image: mi-app:2.1.0
      command: ["sh", "-c", "source /vault/secrets/db-creds && source /vault/secrets/config && java -jar app.jar"]
```

```bash
# Configurar Vault con el metodo de autenticacion de Kubernetes
vault auth enable kubernetes
vault write auth/kubernetes/config \
  kubernetes_host="https://kubernetes.default.svc" \
  token_reviewer_jwt="$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
  kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt

# Crear un rol de Vault vinculado a un service account
vault write auth/kubernetes/role/app-production-role \
  bound_service_account_names=app-service-account \
  bound_service_account_namespaces=produccion \
  policies=app-production-policy \
  ttl=1h
```

> Vault soporta多种 tipos de mecanismos de rotacion automatica: con TTLs cortos que generan credenciales temporales, con webhooks que notifican a la aplicacion, o con sidecar containers que refrescan los secrets periodicamente. La estrategia de rotacion debe adaptarse a los requisitos de disponibilidad de cada servicio.

## Buena practica: gestor externo como Vault

En entornos productivos, la recomendacion es no almacenar secrets directamente en Kubernetes, sino usar un gestor externo que proporcione:

1. Cifrado en reposo con gestiones de claves propias
2. Control de acceso basado en politicas (RBAC)
3. Auditoria centralizada de accesos
4. Rotacion automatica de credenciales
5. Integración con multiples plataformas (AWS, Azure, GCP)

```yaml
# Ejemplo de ExternalSecret (con External Secrets Operator)
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: app-external-secret
  namespace: produccion
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-backend
    kind: SecretStore
  target:
    name: app-vault-secrets
  data:
    - secretKey: database-password
      remoteRef:
        key: secret/data/app/production
        property: DATABASE_PASSWORD
    - secretKey: api-key
      remoteRef:
        key: secret/data/app/production
        property: API_SECRET_KEY
```

