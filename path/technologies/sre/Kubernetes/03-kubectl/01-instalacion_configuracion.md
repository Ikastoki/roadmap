# Instalacion y configuracion de kubectl

kubectl es la herramienta de linea de comandos que actua como interfaz principal entre el operador y el API server de Kubernetes. Sin ella, cualquier interaccion con el cluster requiere llamadas HTTP directas contra la API REST, algo practico para scripts pero inviable para el trabajo diario. El nombre es una contraccion de _Kubernetes control_, y su funcion es traducir comandos legibles por humanos en peticiones JSON/YAML contra el API server, manejando autenticacion, formato de salida y gestion de errores.

Instalar kubectl no es suficiente: la herramienta necesita saber a que cluster conectarse, con que credenciales y en que contexto operar. Esa informacion se almacena en el archivo `~/.kube/config`, el corazon de la configuracion de kubectl.

## Instalacion en Linux

La forma mas directa en Linux es descargar el binario oficial y colocarlo en el PATH. Kubernetes publica versiones estables etiquetadas semver, y el binario incluye todas las dependencias necesarias, por lo que no requiere paquetes adicionales.

```bash
# Descargar la ultima version estable
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

# Dar permisos de ejecucion
chmod +x kubectl

# Mover a un directorio del PATH
sudo mv kubectl /usr/local/bin/

# Verificar la instalacion
kubectl version --client
```

La descarga del binario no depende de ningun gestor de paquetes, lo que la hace consistente entre distribuciones. Sin embargo, distribuciones como Ubuntu, Debian, Fedora o Arch Linux ofrecen kubectl en sus repositorios oficiales, aunque con versiones que pueden no estar actualizadas.

```bash
# Alternativa con gestor de paquetes (Debian/Ubuntu)
sudo apt-get update && sudo apt-get install -y kubectl

# Alternativa con snap
sudo snap install kubectl --classic
```

> La descarga directa del binario desde dl.k8s.io es la opcion recomendada por la documentacion oficial. Los paquetes de distribucion pueden retrasarse varias versiones, y kubectl suele mantener compatibilidad hacia atras con la version anterior del API server (n-1), pero no con versiones mucho mas antiguas.

## Instalacion en macOS

En macOS existen dos vias principales: Homebrew o descarga directa del binario. Homebrew gestiona automaticamente las actualizaciones, mientras que la descarga directa ofrece la ultima version inmediatamente.

```bash
# Con Homebrew
brew install kubectl

# Verificar
kubectl version --client
```

La instalacion mediante Homebrew instala kubectl junto con sus dependencias (como bash-completion para autocompletado). Si se prefiere la descarga directa, el proceso es identico al de Linux pero con la arquitectura adecuada:

```bash
# Para macOS Intel
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/darwin/amd64/kubectl"

# Para Apple Silicon
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/darwin/arm64/kubectl"
```

## Instalacion en Windows

Windows ofrece tres metodos: el binario nativo, el gestor Chocolatey o Docker Desktop (que incluye kubectl integrado).

```powershell
# Con Chocolatey
choco install kubernetes-cli

# Con winget
winget install Kubernetes.kubectl
```

La descarga directa requiere determinar la arquitectura adecuada y anadir el directorio al PATH manualmente:

```powershell
# Descargar el binario desde dl.k8s.io y guardarlo en C:\kubectl
# Anadir C:\kubectl al PATH del sistema desde "Variables de entorno"
```

## Autocompletado en la terminal

kubectl proporciona autocompletado para bash, zsh y fish. Activar esta funcionalidad ahorra errores de tipeo y acelera el trabajo diario.

```bash
# Para bash
source <(kubectl completion bash)
echo "source <(kubectl completion bash)" >> ~/.bashrc

# Para zsh
source <(kubectl completion zsh)
echo "source <(kubectl completion zsh)" >> ~/.zshrc
```

## El archivo kubeconfig

El archivo `~/.kube/config` es un fichero YAML que almacena toda la informacion necesaria para que kubectl se conecte a uno o varios clusters. No es un archivo de configuracion global de Kubernetes, sino especifico del cliente. Cada usuario que ejecuta kubectl tiene su propio kubeconfig, lo que permite que diferentes operadores tengan accesos y contextos distintos.

La estructura del kubeconfig se organiza en tres secciones principales:

```yaml
apiVersion: v1
kind: Config
clusters:
- cluster:
    certificate-authority-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0t...
    server: https://192.168.1.100:6443
  name: my-cluster
contexts:
- context:
    cluster: my-cluster
    namespace: default
    user: admin
  name: admin@my-cluster
current-context: admin@my-cluster
users:
- name: admin
  user:
    client-certificate-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0t...
    client-key-data: LS0tLS1CRUdJTiBSU0EgUFJJVkFURSBLRVktLS0tLQ...
```

**Clusters**: define los endpoints del API server y la autoridad de certificacion (CA) que firma el certificado TLS del servidor. Cada entrada incluye la URL del API server y el material criptografico necesario para verificar la identidad del servidor.

**Users**: almacena las credenciales de los usuarios o service accounts. Pueden ser certificados de cliente (mutual TLS), tokens estaticos, o referencias a proveedores externos como exec plugins (aws-iam-authenticator, gcloud, azure CLI).

**Contexts**: asocia un cluster, un usuario y opcionalmente un namespace. El context es la tripleta que define el destino de cada operacion: a que cluster, con que credenciales y en que namespace por defecto.

> kubectl permite usar multiples kubeconfigs simultaneamente mediante la variable de entorno `KUBECONFIG`. Si esta variable esta definida, kubectl fusiona todos los archivos que apunta siguiendo reglas de precedencia. Esto es util en entornos con multiples clusters (desarrollo, staging, produccion) sin necesidad de un unico archivo monolítico.

## Gestion de contextos

El campo `current-context` determina que context esta activo. kubectl envia todas las peticiones al cluster y con las credenciales definidas en ese context. Cambiar de context equivale a cambiar de cluster o de usuario sin necesidad de reconfigurar nada.

```bash
# Ver contextos configurados
kubectl config get-contexts

# Cambiar al context activo
kubectl config use-context admin@my-cluster

# Ver el context actual
kubectl config current-context
```

## Verificar la conexion

Una vez configurado el kubeconfig, el comando mas basico para verificar que kubectl puede comunicarse con el API server es `cluster-info`:

```bash
kubectl cluster-info
```

Este comando muestra las URLs de los servicios del control plane (API server, CoreDNS, dashboard si esta instalado, etc.). Si el API server responde, la conexion esta funcionando. Si no, el mensaje de error suele indicar si el problema es de red, certificados o credenciales.

```bash
# Version detallada del cliente y del servidor
kubectl version

# El flag --kubeconfig permite usar un archivo alternativo
kubectl --kubeconfig /ruta/alternativa/config cluster-info
```

> `kubectl cluster-info` no verifica que el cluster tenga nodos funcionales o que los pods se ejecuten correctamente. Solo confirma que kubectl llega al API server y que este responde. Un cluster puede responder a cluster-info y aun asi tener todos los nodos en estado NotReady.

## Archivo kubeconfig por defecto

Si no existe `~/.kube/config`, kubectl no puede conectarse a ningun cluster. La mayoria de herramientas de instalacion de Kubernetes (kubeadm, minikube, kind, Rancher, plataformas cloud) generan automaticamente este archivo durante el proceso de instalacion y lo colocan en la ruta esperada.

```bash
# La estructura de directorios se crea automaticamente
# al ejecutar cualquier comando kubectl que necesite config
ls -la ~/.kube/
# total 12
# drwxr-xr-x  3 user user 4096 ...
# -rw-------  1 user user  789 ... config
```

El archivo contiene datos sensibles (certificados, tokens), por lo que Kubernetes recomienda permisos restrictivos: `chmod 600 ~/.kube/config`. Cualquier usuario con acceso de lectura a este archivo puede ejecutar comandos en el cluster con los permisos asociados.

