# Cuando usar que

La proliferation de tecnologias de aislamiento y ejecucion (VMs, contenedores, sandbox, serverless) puede generar confusion. Cada tecnologia tiene un escenario ideal de uso, y elegir la incorrecta resulta en ineficiencia, costos innecesarios o problemas de seguridad. La clave esta en entender las dimensiones de decision: aislamiento, rendimiento, portabilidad, complejidad operativa y costo.

## Matriz de decision

```
Aislamiento maximo ──────────────────► Aislamiento minimo
     │                                        │
     ▼                                        ▼
   VM fisica   VM en nube   Contenedor   Sandbox   Serverless
   (dedicado)  (IaaS)       (Docker)     (Firejail) (Lambda)
```

## Escenarios y recomendacion

**VM (maquina virtual):**
- Necesitas ejecutar Windows y Linux en el mismo servidor.
- Software legacy que requiere un SO especifico.
- Multi-tenancy donde el aislamiento entre clientes es critico.
- Entornos de produccion con requisitos de seguridad estrictos.
- Cargas de trabajo que necesitan control total del SO.

```bash
# Ejemplo: servidor Windows IIS + servidor Linux Apache
# en el mismo hardware fisico
VM 1: Windows Server 2022 + IIS (aplicacion legacy .NET)
VM 2: Ubuntu 22.04 + Apache (aplicacion web moderna)
```

**Contenedor (Docker/Podman):**
- Microservicios que se despliegan frecuentemente.
- CI/CD con entornos efimeros (crear/destruir en minutos).
- Aplicaciones cloud-native disenadas para contenedores.
- Maximizar densidad de aplicaciones en un servidor.
- Desarrollo local replicable (el mismo contenedor corre en cualquier maquina).

```bash
# Ejemplo: arquitectura de microservicios
docker compose up -d
# Contenedor 1: API REST (Node.js)
# Contenedor 2: Base de datos (PostgreSQL)
# Contenedor 3: Cache (Redis)
# Contenedor 4: Proxy inverso (Nginx)
```

**Serverless (Lambda/Functions):**
- Funciones que se ejecutan en respuesta a eventos.
- API endpoints con trafico variable o impredecible.
- Procesamiento de archivos subidos a un bucket.
- Bots, webhooks, notificaciones.
- Prototipos rapidos sin infraestructura.

```bash
# Ejemplo: API que escala a cero
# Lambda se activa solo cuando recibe peticiones
# No hay servidor mantenido
# Pago: solo por los milisegundos de ejecucion
```

**Sandbox (Firejail/seccomp):**
- Ejecutar codigo de fuentes no confiables.
- Navegacion web en entornos no confiables.
- Aislar procesos individuales sin overhead de VM o contenedor.
- Pruebas automatizadas que no deben afectar el sistema.

## Combinacion de tecnologias

En la practica, rara vez se usa una sola tecnologia. La combinacion mas comun es:

```
Servidor fisico o instancia en nube
  └── VM (para aislamiento de SO)
        └── Contenedor (para aislamiento de aplicacion)
              └── Sandbox (para aislar procesos internos)
```

Kubernetes se despliega sobre VMs en la nube; las aplicaciones corren en contenedores dentro de Kubernetes; y algunos procesos internos pueden usar sandbox adicionales.

## Regla simple

- **Necesitas otro SO?** VM.
- **Necesitas aislar una aplicacion?** Contenedor.
- **Necesitas ejecutar algo temporal?** Serverless.
- **Necesitas aislar un proceso?** Sandbox.
- **No sabes?** Empieza con contenedores y escala a VMs si es necesario.

> No existe la tecnologia perfecta; existe la tecnologia adecuada para cada caso. Un equipo que conoce las fortalezas y debilidades de cada opcion puede disenar una infraestructura que sea eficiente, segura y economica. La clave no es elegir una sola tecnologia, sino combinarlas correctamente.

