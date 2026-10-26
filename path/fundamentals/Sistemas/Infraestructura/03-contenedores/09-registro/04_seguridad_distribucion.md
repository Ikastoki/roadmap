# Seguridad en la distribucion

La seguridad en la distribucion de imagenes garantiza que las imagenes que se ejecutan en produccion son autenticas, integras y libres de vulnerabilidades conocidas. El firmado de imagenes verifica la procedencia, y el escaneo de vulnerabilidades detecta problemas antes de que lleguen a produccion.

## Firmado de imagenes

```bash
# Docker Content Trust (DCT)
export DOCKER_CONTENT_TRUST=1

# Push con firma
docker push miusuario/mi-app:1.0
# La primera vez genera claves de firma
# Las siguientes veces firma automaticamente

# Verificar firma
docker pull miusuario/mi-app:1.0
# Verifica la firma antes de descargar
```

```bash
# Cosign (herramienta moderna)
# Instalar
go install github.com/sigstore/cosign/cmd/cosign@latest

# Firmar imagen
cosign sign --key cosign.key docker.io/miusuario/mi-app:1.0

# Verificar firma
cosign verify --key cosign.pub docker.io/miusuario/mi-app:1.0
```

## Escaneo de vulnerabilidades

```bash
# Trivy (herramienta open source)
# Instalar
sudo apt-get install trivy

# Escanear imagen
trivy image nginx:alpine
# Total: 15 (UNKNOWN: 0, LOW: 8, MEDIUM: 5, HIGH: 2, CRITICAL: 0)

# Escanear con salida JSON
trivy image -f json -o report.json nginx:alpine

# Escanear en pipeline
trivy image --exit-code 1 --severity HIGH,CRITICAL nginx:alpine
# Falla si hay vulnerabilidades HIGH o CRITICAL
```

```bash
# Docker Scout
docker scout cves nginx:alpine
docker scout recommendations nginx:alpine
```

```bash
# Snyk
snyk container test nginx:alpine
```

## Escaneo en CI/CD

```yaml
# GitHub Actions
name: Scan Image
on: push

jobs:
  scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Build image
        run: docker build -t mi-app:${{ github.sha }} .
      - name: Scan with Trivy
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: mi-app:${{ github.sha }}
          format: table
          exit-code: 1
          severity: HIGH,CRITICAL
```

## Politicas de seguridad

```bash
# No ejecutar imagenes sin verificar
docker pull miusuario/mi-app:1.0  # Verifica firma si DCT esta habilitado

# Usar imagenes minimas
# Alpine, Distroless, Scratch tienen menos superficie de ataque

# Actualizar imagenes regularmente
docker pull nginx:alpine  # Siempre la ultima version
```

> La seguridad en la distribucion no es opcional: cada imagen que se ejecuta en produccion debe ser verificada (firma) y escaneada (vulnerabilidades). Integrar el escaneo de vulnerabilidades en el pipeline de CI/CD garantiza que los problemas se detecten antes de que lleguen a produccion.

