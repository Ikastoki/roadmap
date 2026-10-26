# systemd: el sistema de init moderno

La gestión de servicios en Linux ha evolucionado significativamente desde los scripts de shell que ejecutaba SysVinit en orden de secuencia lineal. systemd surge como reemplazo moderno con un enfoque basado en un grafo de dependencias paralelo, permitiendo que los servicios se inicien de forma concurrente siempre que sus dependencias lo permitan. Esto reduce drásticamente el tiempo de arranque del sistema y ofrece una gestión más robusta de los procesos demonio.

## Por que systemd reemplazó a SysVinit

SysVinit ejecutaba scripts de init en un orden determinado por directorios numerados (`/etc/rc0.d/`, `/etc/rc1.d/`, etc.). Este modelo presentaba varios problemas:

- **Inicio secuencial obligatorio**: cada servicio debía esperar a que el anterior terminara completamente, generando cuellos de botella.
- **Scripts de shell no estandarizados**: cada distribución implementaba sus propias convenciones, dificultando la portabilidad.
- **Gestion manual de procesos**: si un demonio moría inesperadamente, SysVinit no lo reiniciaba automáticamente.
- **Falta de introspeccion**: no existia una forma uniforme de consultar el estado de un servicio.

systemd resuelve estos problemas mediante un daemon centralizado (PID 1) que gestiona todo el ciclo de vida de los servicios, registra logs integrados y proporciona una interfaz unificada (`systemctl`) para interactuar con ellos.

## Unidades y tipos principales

El concepto central de systemd es la **unidad** (unit), que representa un recurso o componente que el sistema puede gestionar. Cada unidad se define en un archivo `.service`, `.timer`, `.socket` o similar, y contiene la configuracion de como iniciar, detener y monitorizar ese recurso.

### Unidades de servicio (`.service`)

Son las mas comunes. Definen un demonio o proceso que se ejecuta en segundo plano. Un archivo de servicio tipico contiene secciones como `[Unit]`, `[Service]` y `[Install]`:

```ini
[Unit]
Description=Mi aplicacion web
After=network.target postgresql.service
Requires=postgresql.service

[Service]
Type=simple
User=www-data
ExecStart=/usr/bin/mi-app --config /etc/mi-app.conf
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Las directivas clave son `After` y `Requires`, que establecen las dependencias: el servicio solo arrancara despues de que `postgresql.service` este activo y fallara si este no esta disponible.

### Unidades de temporizador (`.timer`)

Sustituyen a cron para tareas programadas. Un temporizador define **cuando** ejecutar algo, y apunta a un servicio correspondiente que define **que** ejecutar:

```ini
[Unit]
Description=Ejecutar limpieza diaria

[Timer]
OnCalendar=daily
Persistent=true

[Install]
WantedBy=timers.target
```

La ventaja sobre cron es que las ejecuciones perdidas (por ejemplo, si el servidor estaba apagado) se recuperan cuando se usa `Persistent=true`.

### Unidades de socket (`.socket`)

Implementan la activacion bajo demanda. En lugar de mantener un servicio permanentemente en memoria, systemd abre un socket y solo lanza el servicio cuando recibe una peticion en ese socket. Esto reduce el consumo de recursos en servicios que no se usan continuamente.

## Comandos esenciales de systemctl

`systemctl` es la herramienta de linea de comandos para interactuar con systemd. Las operaciones mas frecuentes son:

| Comando | Funcion |
|---|---|
| `systemctl start <unidad>` | Inicia un servicio inmediatamente |
| `systemctl stop <unidad>` | Detiene un servicio |
| `systemctl restart <unidad>` | Detiene y vuelve a iniciar el servicio |
| `systemctl reload <unidad>` | Recarga la configuracion sin reiniciar |
| `systemctl status <unidad>` | Muestra el estado actual y las ultimas lineas de log |
| `systemctl enable <unidad>` | Marca el servicio para iniciar automaticamente en el arranque |
| `systemctl disable <unidad>` | Elimina el servicio del arranque automatico |
| `systemctl is-active <unidad>` | Comprueba si un servicio esta ejecutandose (devuelve 0 o 1) |

La diferencia entre `restart` y `reload` es fundamental: `restart` mata el proceso y lo vuelve a lanzar, mientras que `reload` envia la signal `SIGHUP` al proceso para que relee su configuracion sin perder conexiones activas.

## Gestion de dependencias entre servicios

systemd construye un grafo de dependencias entre todas las unidades del sistema. Existen dos tipos de relaciones:

- **Requires**: la unidad dependiente necesita que la otra este activa. Si la dependencia falla, la unidad dependiente tambien falla.
- **Wants**: la unidad dependiente desea que la otra este activa, pero no falla si esta no arranca.

A nivel de ordenamiento, `After` y `Before` controlan la secuencia de inicio. Un servicio con `After=network.target` esperara a que la red este configurada antes de arrancar.

Esto permite crear escenarios complejos: un servidor web que necesita que la base de datos este activa, que a su vez necesita que el sistema de archivos este montado, todo coordinado automaticamente por el grafo de dependencias.

> En entornos de produccion, es recomendable usar `systemctl list-dependencies <unidad>` para inspeccionar el arbol completo de dependencias antes de modificar cualquier servicio critico.

