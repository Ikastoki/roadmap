# Gestion de Imagenes Seguras

Las imágenes de contenedor son la unidad fundamental de despliegue en Kubernetes, y cada una contiene un sistema de archivos completo con dependencias, bibliotecas compartidas y binarios del sistema. Una vulnerabilidad en cualquiera de estos componentes puede ser explotada para comprometer no solo el contenedor, sino el nodo y potencialmente el clúster entero. La gestion segura de imágenes implica un pipeline que abarca desde el escaneo de vulnerabilidades en tiempo de construcción hasta la verificación criptográfica de la integridad de las imágenes antes de su ejecución.

## Escaneo de vulnerabilidades

El escaneo de vulnerabilidades analiza las capas de una imagen contra bases de datos de vulnerabilidades conocidas (CVE). Herramientas como Trivy, Snyk y Clair comparan los paquetes instalados en la imagen con los registros de vulnerabilidades publicados por los distribuidores de software y proyectos de código abierto. Cada vulnerabilidad recibe una clasificación de severidad que determina si la imagen puede desplegarse o no.

```bash
# Escaneo basico con Trivy
trivy image nginx:1.25

# Escaneo con salida en formato JSON para integracion en pipelines
trivy image --format json --output trivy-report.json nginx:1.25

# Escaneo con severidad minima: CRITICAL y HIGH
trivy image --severity CRITICAL,HIGH --exit-code 1 nginx:1.25

# Escaneo de un directorio con Dockerfile
trivy config --severity CRITICAL,HIGH ./kubernetes/

# Escaneo de un grafico Helm
trivy helm --severity CRITICAL,HIGH ./mi-grafico/
```

> El flag `--exit-code 1` hace que Trivy devuelva un código de error no cero cuando encuentra vulnerabilidades que coinciden con la severidad especificada. Esto permite detener automaticamente un pipeline de CI/CD si la imagen no supera el escaneo.

```bash
# Escaneo con Snyk
snyk container test nginx:1.25 --severity-threshold=high

# Escaneo continuo de imagenes en un registro
snyk container monitor nginx:1.25 --file=Dockerfile --project-name=mi-app
```

```yaml
# Ejemplo de pipeline de escaneo en GitHub Actions
name: Escaneo de seguridad
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  escaneo:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Construir imagen
        run: docker build -t mi-app:${{ github.sha }} .
      - name: Escaneo con Trivy
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: mi-app:${{ github.sha }}
          format: sarif
          output: trivy-results.sarif
          severity: CRITICAL,HIGH
          exit-code: 1
      - name: Subir resultados
        uses: github/codeql-action/upload-sarif@v3
        if: always()
        with:
          sarif_file: trivy-results.sarif
```

> Clair es otro escaneador popular que opera como un servicio centralizado. A diferencia de Trivy (que es un CLI), Clair almacena las vulnerabilidades en una base de datos y expone una API para consultas. Es más adecuado para entornos donde múltiples servicios necesitan consultar vulnerabilidades de forma centralizada.

## Firma y verificacion de imagenes

La firma de imágenes garantiza la integridad y la autenticidad de las imágenes que se ejecutan en el clúster. Cosign (parte del proyecto Sigstore) permite firmar imágenes de contenedor criptográficamente y verificar esas firmas antes de su ejecución. Notary v2 proporciona un marco similar basado en estándares de firma de software.

```bash
# Instalar Cosign
go install github.com/sigstore/cosign/v2/cmd/cosign@latest

# Generar par de claves para firma
cosign generate-key-pair

# Firmar una imagen (requiere autenticacion con el registro)
cosign sign --key cosign.key registry.empresa.com/mi-app:v1.0

# Verificar la firma de una imagen
cosign verify --key cosign.pub registry.empresa.com/mi-app:v1.0

# Firmar con identity y issuer (firma sin clave con OIDC)
cosign sign --identity-token $OIDC_TOKEN registry.empresa.com/mi-app:v1.0

# Verificar firma sin clave
cosign verify \
  --certificate-identity=juan@empresa.com \
  --certificate-oidc-issuer=https://auth.empresa.com \
  registry.empresa.com/mi-app:v1.0
```

> La firma sin clave (keyless) utiliza identidades OIDC (como Google, GitHub o un IdP corporativo) para vincular la firma a una identidad verificable. Esto elimina la necesidad de gestionar claves criptográficas y permite verificar quién firmó una imagen basándose en su correo electrónico o ID de servicio.

```yaml
# Politica de verificacion de firmas con Kyverno
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: verificar-firma
spec:
  validationFailureAction: Enforce
  background: false
  rules:
    - name: verificar-cosign
      match:
        any:
          - resources:
              kinds:
                - Pod
      verifyImages:
        - imageReferences:
            - "registry.empresa.com/*"
          attestors:
            - count: 1
              entries:
                - keys:
                    publicKeys: |-
                      -----BEGIN PUBLIC KEY-----
                      MFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAE...
                      -----END PUBLIC KEY-----
                  rekor:
                    url: https://rekor.sigstore.dev
```

## Admission policies para imagenes

La capa final de seguridad consiste en validar las imágenes en el momento de su despliegue mediante un admission controller. Esto garantiza que solo se ejecuten imágenes que provengan de registros aprobados, que hayan pasado el escaneo de vulnerabilidades y que estén firmadas criptograficamente.

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: politica-imagenes-seguras
spec:
  validationFailureAction: Enforce
  background: true
  rules:
    - name: registros-permitidos
      match:
        any:
          - resources:
              kinds:
                - Pod
      validate:
        message: "Las imagenes solo pueden provenir de registros corporativos aprobados."
        pattern:
          spec:
            containers:
              - image: "registry.empresa.com/* | gcr.io/proyecto-empresa/*"
    - name: sin-latest
      match:
        any:
          - resources:
              kinds:
                - Pod
      validate:
        message: "No se permite usar la etiqueta 'latest' en imagenes."
        pattern:
          spec:
            containers:
              - image: "!*:latest"
    - name: scaneo-obligatorio
      match:
        any:
          - resources:
              kinds:
                - Pod
      verifyImages:
        - imageReferences:
            - "registry.empresa.com/*"
          attestors:
            - entries:
                - attestor: "juan@empresa.com"
                  issuer: "https://auth.empresa.com"
```

```bash
# Verificar que una imagen esta en el registro aprobado y tiene firma
cosign verify \
  --certificate-identity="juan@empresa.com" \
  --certificate-oidc-issuer="https://auth.empresa.com" \
  registry.empresa.com/mi-app:v1.2.3

# Inspeccionar las capas de una imagen para entender su composicion
crane manifest registry.empresa.com/mi-app:v1.2.3

# Listar las vulnerabilidades almacenadas junto a la imagen (si se uso SBOM)
cosign verify-attestation \
  --type spdxjson \
  --key cosign.pub \
  registry.empresa.com/mi-app:v1.2.3
```

> Un SBOM (Software Bill of Materials) es un inventario completo de todas las dependencias y componentes de software que contiene una imagen. Cosign permite adjuntar SBOMs como attestations a las firmas, proporcionando trazabilidad completa desde el código fuente hasta la imagen desplegada en producción.

