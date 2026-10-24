# OpenCode CLI

## GitHub Actions

La integracion de OpenCode con GitHub Actions materializa el salto desde un agente local de codigo hacia un participante automatico en los flujos de integracion continua del equipo. Cuando un desarrollador ejecuta `opencode github install`, el comando genera un archivo `.github/workflows/opencode.yml` en el repositorio, configurando un workflow completo que permite a OpenCode operar dentro del entorno de CI. Este workflow se activa tipicamente ante eventos de pull request, aunque puede configurarse para responder a pushes, comentarios o eventos programados. El flujo interno del workflow realiza el checkout del codigo, instala OpenCode, configura la autenticacion mediante un token de GitHub, y ejecuta `opencode github run` con los parametros del evento que desencadeno la ejecucion.

La instruccion `opencode github run` detecta automaticamente el contexto de CI en el que se ejecuta: extrae el diff del pull request, identifica los archivos modificados, accede al historial de commits de la rama y presenta al agente un entorno de trabajo completo con el codigo fusionado. Sobre ese contexto, OpenCode puede realizar revisiones de codigo automatizadas analizando cada cambio en busca de problemas de estilo, vulnerabilidades de seguridad, patrones antiestandar, o desviaciones de las convenciones del proyecto definidas en `AGENTS.md`. Los resultados se publican como comentarios en el hilo del pull request, con referencias precisas a las lineas afectadas, sugerencias de correccion y, cuando la configuracion de permisos lo permite, commits correctivos directamente sobre la rama.

```yaml
name: OpenCode CI Review
on:
  pull_request:
    types: [opened, synchronize, reopened]
  push:
    branches: [main]

permissions:
  contents: write
  pull-requests: write
  issues: write

jobs:
  review:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: "20"

      - name: Install OpenCode
        run: npm install -g @opencode/cli

      - name: Run OpenCode agent
        run: opencode github run --event pull_request --token "${{ secrets.GITHUB_TOKEN }}"
        env:
          OPENCODE_API_KEY: ${{ secrets.OPENCODE_API_KEY }}
          OPENCODE_PROVIDER: anthropic
```

La potencia de este modelo reside en que OpenCode opera dentro del CI con las mismas capacidades que en una sesion local: acceso completo al arbol de directorios, ejecucion de comandos, invocacion de herramientas MCP y modificacion de archivos. La diferencia es que el contexto de trabajo es el diff del PR, y las acciones del agente quedan registradas en el propio repositorio a traves de la API de GitHub. Esto permite establecer politicas de calidad automaticas donde ningun PR puede fusionarse sin haber pasado por una revision del agente, o donde el agente actua como un segundo revisor que complementa la supervision humana.

> El workflow generado por `opencode github install` es personalizable. Los equipos pueden modificar el disparador del evento, anadir pasos previos como la instalacion de dependencias del proyecto, o configurar la ejecucion de tests antes o despues de la intervencion del agente. La unica condicion es que la autenticacion via token de GitHub este correctamente configurada para que el agente pueda publicar sus resultados.

## Modo servidor

OpenCode puede operar como un servidor headless mediante el comando `opencode serve`, exponiendo toda su funcionalidad a traves de una API HTTP. En esta modalidad, el agente no requiere una terminal interactiva ni un usuario conectado: escucha en un puerto configurable, recibe peticiones con prompts en formato JSON y devuelve respuestas estructuradas. Esta arquitectura permite integrar OpenCode en pipelines de CI/CD personalizados, herramientas de terceros, paneles de administracion, o cualquier sistema capaz de realizar peticiones HTTP. El modo servidor es particularmente util en entornos donde multiples sistemas necesitan acceder al agente de forma concurrente sin compartir una sesion de terminal.

El comando `opencode web` activa una interfaz grafica accesible desde el navegador, ideal para entornos de equipo donde los desarrolladores prefieren no trabajar en la terminal. Esta interfaz replica las capacidades del modo TUI pero en un formato web, con paneles de conversacion, historial de sesiones y controles de configuracion. Por su parte, `opencode attach` permite conectar una sesion TUI local a un servidor remoto: un desarrollador puede ejecutar OpenCode en su maquina local pero dirigir las operaciones del agente hacia un servidor OpenCode que se ejecuta en una maquina remota, evitando tener que instalar el agente en entornos aislados o con recursos limitados.

La autenticacion basica via HTTP protege el acceso al servidor. `opencode serve` acepta un flag `--auth` con un par usuario:contraseña, o la configuracion de un token Bearer. En entornos de produccion, es recomendable situar el servidor tras un proxy reverso (nginx, Caddy) que gestione TLS, autenticacion adicional y limitacion de tasa.

```bash
# Iniciar servidor headless en el puerto 8080
opencode serve --port 8080 --auth admin:secreto

# Conectar una sesion TUI local a un servidor remoto
opencode attach --server https://opencode.empresa.com --token m1-t0k3n

# Abrir la interfaz web
opencode web --port 3000
```

Las arquitecturas de despliegue para el modo servidor varian segun el ambito. Un servidor monousuario es la configuracion mas simple: OpenCode se ejecuta como un servicio systemd en una maquina de desarrollo, accesible solo desde esa misma maquina o desde la red local del desarrollador. Un servidor compartido para equipos implica exponer OpenCode en una red interna con autenticacion, permitiendo que varios miembros del equipo ejecuten sesiones independientes contra el mismo agente, cada una con su propio contexto y su propio historial. En entornos enterprise, el servidor se situa detras de un proxy corporativo que centraliza la autenticacion (LDAP, SAML, OIDC), el registro de auditoria y las politicas de acceso, actuando como un gateway que controla que usuarios pueden invocar al agente y bajo que restricciones.

> El modo servidor no mantiene estado entre peticiones a menos que se configure una sesion persistente. Cada peticion HTTP puede especificar un identificador de sesion en la cabecera `X-Session-Id` para reanudar conversaciones anteriores. Sin este identificador, cada llamada crea una sesion efimera que se descarta al completar la respuesta.

## Enterprise

OpenCode ofrece mecanismos de gestion centralizada para entornos corporativos donde es necesario imponer politicas de uso, restringir el acceso a modelos, o garantizar el cumplimiento normativo. La configuracion gestionada via MDM (Mobile Device Management) permite a los administradores de sistemas distribuir y forzar ajustes en todos los equipos de la organizacion sin intervencion de los usuarios. En macOS, esto se materializa mediante perfiles de configuracion (archivos `.mobileconfig`) que definen los valores de `opencode.json`; en Linux, a traves de archivos de configuracion del sistema en `/etc/opencode/`; en Windows, mediante directivas de grupo (Group Policy Objects). Cuando OpenCode se inicia, carga primero estas configuraciones gestionadas, que actuan como una capa base inmutable antes de aplicar las configuraciones del usuario.

El mecanismo de configuracion remota via `/.well-known/opencode` permite que servidores HTTP proporcionen configuracion organizacional sin necesidad de MDM. Cuando OpenCode arranca, puede consultar un dominio configurado (por ejemplo, `https://opencode.miempresa.com/.well-known/opencode`) y descargar un JSON con directrices que se fusionan con la configuracion local. Este enfoque es util para organizaciones que no pueden o no quieren gestionar dispositivos via MDM, pero necesitan mantener politicas centralizadas como la URL del proxy corporativo, los proveedores de modelo autorizados o el endpoint del servidor MCP de auditoria.

La precedencia de configuracion en entornos enterprise sigue una jerarquia estricta. Las configuraciones gestionadas (via MDM, archivos de sistema, o `.well-known`) residen en el nivel mas alto y no pueden ser sobrescritas por configuraciones de usuario ni por parametros de linea de comandos. Por debajo se situa la configuracion del proyecto (`opencode.json` local), seguida de la configuracion del usuario (`~/.config/opencode/opencode.json`), y finalmente los flags de linea de comandos que tienen la menor precedencia pero pueden sobrescribir cualquier nivel no gestionado. Esta jerarquia permite a las organizaciones establecer limites de seguridad (por ejemplo, "solo el modelo Claude de Anthropic via AWS Bedrock") mientras los desarrolladores conservan autonomia para configurar aspectos no restringidos como el color scheme o el nivel de verbose.

Los casos de uso enterprise mas habituales incluyen la imposicion del uso de una API key corporativa para evitar fugas de credenciales, la restriccion del catalogo de modelos a aquellos aprobados por el departamento de seguridad, la activacion forzosa del registro de auditoria que captura todas las interacciones con el agente y las envia a un servidor MCP centralizado, y la configuracion de limites de gasto maximos por sesion o por dia para controlar el coste operativo. En sectores regulados como banca, sanidad o defensa, estas capacidades permiten desplegar agentes de codigo en entornos que de otro modo no podrian adoptar herramientas de inteligencia artificial por restricciones de compliance.

```json
{
  "managed": {
    "provider": "anthropic",
    "model": "claude-sonnet-4-20250514",
    "apiKey": "{env:OPENCODE_ENTERPRISE_KEY}",
    "auditLog": {
      "enabled": true,
      "endpoint": "https://audit.miempresa.com/opencode",
      "includePrompts": true
    },
    "rateLimit": {
      "maxDailyCost": 10.00,
      "maxSessionCost": 2.00
    },
    "allowedProviders": ["anthropic", "bedrock"],
    "mcpServers": {
      "corporate-db": {
        "type": "remote",
        "url": "https://mcp.internal.corp/db",
        "enforced": true
      }
    }
  }
}
```

> Las configuraciones gestionadas se distinguen de las normales por su ubicacion en el sistema de archivos: los archivos en rutas de sistema (`/etc/opencode/`, `ProgramData/OpenCode/`) tienen prioridad absoluta. OpenCode permite a los administradores verificar que las restricciones se aplican correctamente mediante `opencode config --show-managed`, que muestra exclusivamente los valores provenientes de fuentes gestionadas.

