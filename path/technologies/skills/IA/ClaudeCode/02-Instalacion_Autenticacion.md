# Claude Code CLI

Un terminal sin Claude Code es simplemente un interprete de comandos. Con Claude Code, ese mismo terminal se convierte en un entorno de desarrollo aumentado donde la inteligencia artificial opera como un colaborador directo sobre el codigo, el sistema de ficheros y el ciclo de desarrollo. La instalacion y autenticacion correctas determinan no solo el acceso a la herramienta, sino el modelo de interaccion, los limites de uso y la integracion con servicios corporativos.

## Metodos de instalacion

Claude Code se distribuye a traves de multiples canales, cada uno adaptado a un flujo de trabajo distinto. El metodo recomendado en macOS es Homebrew:

```bash
brew install claude-code
```

Esta formula instala el binario `claude` y lo integra en el sistema como cualquier otra herramienta del ecosistema Homebrew. Para entornos donde Homebrew no esta disponible o se prefiere una instalacion global via Node.js, el paquete npm oficial es la alternativa:

```bash
npm install -g @anthropic-ai/claude-code
```

La instalacion via npm descarga el paquete desde el registro publico de npm y lo vincula globalmente, permitiendo su invocacion desde cualquier directorio. Existe tambien un script de instalacion via curl que detecta automaticamente la plataforma y descarga el binario precompilado adecuado:

```bash
curl -fsSL https://cli.claude.ai/install.sh | sh
```

Para quienes prefieran una aplicacion de escritorio con interfaz grafica, Anthropic ofrece una Desktop App descargable desde claude.com/download, disponible para macOS y Windows. No existe version de escritorio para Linux; en este sistema debe utilizarse exclusivamente la CLI via terminal.

Una vez instalado, la verificacion basica consiste en confirmar la version:

```bash
claude --version
```

La actualizacion a la version mas reciente se realiza con el comando interno:

```bash
claude update
```

Si se requiere una version especifica, por ejemplo para mantener compatibilidad con un flujo de CI determinado, puede instalarse directamente indicando el numero de version:

```bash
claude install 2.1.220
```

El modificador `stable` instala la ultima version considerada estable por el equipo de Anthropic, util en entornos donde se prioriza la fiabilidad sobre la novedad.

## Autenticacion

Claude Code requiere una sesion autenticada para operar. El metodo principal de inicio de sesion es el flujo OAuth mediante navegador:

```bash
claude auth login
```

Este comando abre una pagina en el navegador predeterminado donde el usuario introduce sus credenciales de Anthropic. Para entornos donde se desea agilizar el proceso, el flag `--email` pre-rellena el campo de correo electronico:

```bash
claude auth login --email usuario@dominio.com
```

En organizaciones que utilizan inicio de sesion unico (SSO), el flag `--sso` redirige al flujo de autenticacion corporativa:

```bash
claude auth login --sso
```

Para quienes prefieren un modelo de pago por uso sin suscripcion fija, el flag `--console` dirige la autenticacion hacia la API Console de Anthropic, donde el consumo se factura segun el volumen de tokens procesados:

```bash
claude auth login --console
```

Cerrar la sesion activa se realiza con:

```bash
claude auth logout
```

Y para consultar el estado actual de la autenticacion:

```bash
claude auth status
```

Por defecto la salida es en JSON, legible por maquina. Para una representacion pensada para lectura humana se anade `--text`:

```bash
claude auth status --text
```

El modelo de suscripcion de Claude Code se estructura en varios planes. Claude Pro, por 20 dolares mensuales, ofrece acceso al modelo con limites de uso estandar. Claude Max, en su nivel de 100 dolares, incrementa significativamente los limites de contexto y el numero de mensajes. El nivel Claude Max 20x, con un coste de 200 dolares al mes, multiplica por veinte los limites del plan Pro, destinado a equipos que realizan un uso intensivo. La API Console, sin cuota fija, factura exclusivamente por el consumo real de tokens, ideal para integraciones automatizadas y entornos de CI donde el volumen de uso es variable.

> La eleccion del plan no afecta a las capacidades del modelo en si, sino a los limites de cuota de uso. Todos los planes acceden al mismo modelo subyacente, pero con distintos limites de contexto y restricciones de rate limiting.

## Verificacion y diagnostico

Claude Code incorpora herramientas de diagnostico independientes del flujo de trabajo habitual. El comando `doctor` ejecuta una bateria de comprobaciones sobre la instalacion y la configuracion del entorno sin iniciar una sesion interactiva:

```bash
claude doctor
```

Su salida incluye la version del CLI, el estado del binario, la presencia de dependencias del sistema, la configuracion de red y la validez de las credenciales almacenadas. Es la primera herramienta que debe ejecutarse ante cualquier comportamiento inesperado.

Para entornos de integracion continua donde la autenticacion interactiva via navegador no es viable, el CLI permite generar un token OAuth duradero:

```bash
claude setup-token
```

Este comando guia al usuario en la creacion de un token que puede almacenarse como variable de entorno y utilizarse en pipelines automatizados sin intervencion manual.

En ocasiones es necesario eliminar los datos locales que Claude Code ha almacenado para un proyecto concreto, ya sea por corrupcion del cache, cambios en la configuracion o simplemente para empezar desde cero. El comando `project purge` realiza esta limpieza:

```bash
claude project purge --dry-run
```

El flag `--dry-run` muestra que ficheros se eliminarian sin llegar a borrarlos, permitiendo una revision previa. La confirmacion directa se consigue con `-y`:

```bash
claude project purge -y
```

> El purgado de datos locales no afecta al historial remoto ni a las conversaciones almacenadas en los servidores de Anthropic. Solo elimina el cache y los archivos de estado del proyecto en el equipo local.

