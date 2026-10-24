# Contextos y namespaces

En un entorno Kubernetes real no existe un unico cluster. Un ingeniero trabaja simultaneamente con un cluster de desarrollo, uno de staging y uno de produccion, cada uno con sus propias credenciales, configuracion de red y proposito. kubectl gestiona esta multiplicidad mediante contextos: configuraciones empaquetadas que determinan a que cluster, con que usuario y en que namespace por defecto se ejecuta cada operacion.

Los namespaces, por su parte, son la herramienta de segmentacion logica dentro de un mismo cluster. No aïslan a nivel de red ni de seguridad (eso es competencia de Network Policies y RBAC), pero permiten organizar los recursos en agrupaciones semanticas: un namespace para cada equipo, para cada entorno dentro del mismo cluster, o para cada aplicacion.

## Gestion de contextos

Cada contexto en el kubeconfig es una entrada que asocia tres elementos: un cluster, un usuario y un namespace por defecto. El comando `kubectl config` proporciona los subcomandos necesarios para gestionarlos sin editar manualmente el archivo YAML.

```bash
# Listar todos los contextos configurados
kubectl config get-contexts

# Salida tipica
# CURRENT   NAME                CLUSTER         AUTHINFO        NAMESPACE
# *         admin@production    production-cluster  admin       production
#           dev@staging         staging-cluster     developer   staging
#           minikube            minikube        minikube        default
```

El asterisco en la columna `CURRENT` indica el context activo. Cambiar de context es inmediato y no requiere reiniciar nada:

```bash
# Cambiar al context activo
kubectl config use-context admin@production

# Ver el context activo actual
kubectl config current-context
```

El cambio es persistente: kubectl escribe el nuevo `current-context` en el kubeconfig, por lo que cualquier terminal nueva que ejecute kubectl usara ese context.

## Creacion de contextos

Ademas de los contextos generados por herramientas como minikube o kubeadm, se pueden crear manualmente para acceder a clusters adicionales:

```bash
# Anadir un cluster al kubeconfig
kubectl config set-cluster production-cluster \
  --server=https://prod-api.example.com:6443 \
  --certificate-authority=/etc/kubernetes/ca.crt

# Anadir un usuario
kubectl config set-credentials admin \
  --client-certificate=admin.crt \
  --client-key=admin.key

# Crear el contexto que asocia cluster + usuario + namespace
kubectl config set-context admin@production \
  --cluster=production-cluster \
  --user=admin \
  --namespace=production

# Usar el nuevo contexto
kubectl config use-context admin@production
```

Este metodo es especialmente util cuando se reciben archivos kubeconfig de diferentes proveedores. En lugar de sobrescribir el kubeconfig existente, se fusionan las entradas:

```bash
# Fusionar configuraciones usando KUBECONFIG
export KUBECONFIG=~/.kube/config:/path/to/other-config
kubectl config view --flatten > ~/.kube/config
```

> La variable `KUBECONFIG` acepta una lista de rutas separadas por `:` en Linux/macOS o `;` en Windows. kubectl fusiona todos los archivos siguiendo la primera ocurrencia de cada clave. El flag `--flatten` en `kubectl config view` genera un unico archivo con todos los datos embebidos.

## Namespaces

Un namespace es un mecanismo de aislamiento logico dentro del cluster. Los recursos creados en un namespace no son visibles desde otro a menos que se especifique explicitamente, aunque no existe aislamiento de red por defecto.

Kubernetes crea cuatro namespaces durante la instalacion:

```bash
kubectl get namespaces
# NAME              STATUS   AGE
# default           Active   30d
# kube-node-lease   Active   30d
# kube-public       Active   30d
# kube-system       Active   30d
```

- `default`: namespace por defecto para recursos que no especifican uno
- `kube-system`: componentes del control plane y del sistema
- `kube-public`: recursos accesibles sin autenticacion (configmaps publicos)
- `kube-node-lease`: objetos Lease para deteccion de latencia de nodos

```bash
# Listar pods de un namespace especifico
kubectl get pods -n kube-system

# Listar pods de todos los namespaces
kubectl get pods --all-namespaces
# o su alias
kubectl get pods -A
```

## Creacion y eliminacion de namespaces

```bash
# Crear un namespace
kubectl create namespace desarrollo

# Crear un namespace con un archivo YAML
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: produccion
EOF

# Eliminar un namespace (elimina todos los recursos dentro)
kubectl delete namespace desarrollo
```

Eliminar un namespace elimina de forma recursiva todos los recursos contenidos en el. Esto incluye pods, services, deployments, configmaps y secrets. Es una operacion destructiva que debe ejecutarse con precaucion en entornos compartidos.

> Eliminar un namespace puede colgarse si algun recurso dentro tiene finalizers pendientes. En ese caso el namespace queda en estado `Terminating`. La solucion es parchear el namespace para eliminar los finalizers: `kubectl patch namespace <name> -p '{"metadata":{"finalizers":[]}}' --type=merge`.

## Namespace por defecto en un contexto

Cada contexto puede tener un namespace por defecto asociado. Cuando se cambia a ese contexto, kubectl usa ese namespace sin necesidad de especificar `-n` en cada comando.

```bash
# Establecer namespace por defecto en el contexto actual
kubectl config set-context --current --namespace=desarrollo

# Verificar
kubectl config view --minify | grep namespace
```

Si no se especifica namespace en el contexto, kubectl usa `default`. Establecer el namespace correcto en cada contexto evita errores costosos, como desplegar accidentalmente en produccion cuando se cree estar en desarrollo.

## Operaciones con el flag -n

El flag `-n` (o `--namespace`) permite operar en cualquier namespace sin cambiar el contexto ni el namespace por defecto:

```bash
# Operar en un namespace especifico
kubectl get pods -n produccion
kubectl describe service api-gateway -n produccion
kubectl logs -f backend-7c79c4bf97-8kz2h -n produccion

# Operar en todos los namespaces
kubectl get deployments --all-namespaces
```

Este flag es indispensable en entornos compartidos. Permite, por ejemplo, supervisar el namespace de un equipo especifico sin tener que cambiar el context activo, reduciendo el riesgo de operar en el namespace equivocado.

## Buenas practicas con namespaces

Los namespaces no deben usarse como unico mecanismo de aislamiento. La segmentacion real de red se consigue con Network Policies, y el control de acceso con RBAC. Sin embargo, como regla general:

- Un namespace por entorno (dev, staging, prod) dentro del mismo cluster
- Un namespace por equipo si comparten cluster
- Evitar namespaces anidados o jerarquicos (no existen en Kubernetes nativo; requieren Hierarchical Namespace Controller)
- No abusar de namespaces: un cluster con cientos de namespaces puede degradar el rendimiento del API server

```bash
# Ver cuantos recursos hay en cada namespace
kubectl get pods -A | awk '{print $1}' | sort | uniq -c | sort -rn
```

> Los namespaces no aïslan recursos a nivel de cluster como nodes o storage classes. Tampoco aïslan CRDs (Custom Resource Definitions), que son globales. Para aislamiento fuerte entre equipos, considerar clusters separados o herramientas de virtualizacion de clusters como vCluster.

