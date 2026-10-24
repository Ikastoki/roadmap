# WatchService

El servicio de vigilancia de NIO.2 permite a una aplicacion monitorear cambios en directorios del sistema de archivos sin necesidad de realizar sondeos periodicos (polling). Es una implementacion del patron Observer aplicada al sistema de archivos: la aplicacion se suscribe a eventos de creacion, eliminacion o modificacion en un directorio, y el sistema operativo notifica cuando estos ocurren. Internamente, `WatchService` delega en los mecanismos nativos de cada plataforma: `inotify` en Linux, `kqueue` en BSD/macOS y `ReadDirectoryChangesW` en Windows.

## Obtencion y configuracion

Se obtiene una instancia de `WatchService` mediante el metodo `FileSystems.getDefault().newWatchService()`. Luego se registra un directorio en el servicio especificando los tipos de eventos que interesan.

```java
WatchService watchService = FileSystems.getDefault().newWatchService();
Path directorio = Path.of("/home/usuario/monitor");

WatchKey key = directorio.register(watchService,
    StandardWatchEventKinds.ENTRY_CREATE,
    StandardWatchEventKinds.ENTRY_DELETE,
    StandardWatchEventKinds.ENTRY_MODIFY);
```

## Tipos de eventos

`StandardWatchEventKinds` define cuatro tipos de eventos:

- `ENTRY_CREATE` -- se creo un archivo o directorio dentro del directorio vigilado.
- `ENTRY_DELETE` -- se elimino un archivo o directorio.
- `ENTRY_MODIFY` -- se modifico el contenido de un archivo (no aplica a directorios).
- `OVERFLOW` -- indica que algunos eventos se perdieron porque el buffer de eventos del sistema operativo se desbordo. Es un evento especial que debe manejarse para mantener la consistencia.

> `ENTRY_MODIFY` se dispara cuando cambia el contenido del archivo, no cuando cambian sus metadatos (permisos, timestamps). Para monitorear metadatos seria necesario un enfoque diferente basado en atributos.

## WatchKey y el bucle de eventos

`WatchService.poll()` y `WatchService.take()` devuelven objetos `WatchKey` que representan un directorio registrado y los eventos que han ocurrido en el. `take()` es bloqueante: espera hasta que haya eventos. `poll()` es no bloqueante: retorna `null` si no hay eventos pendientes.

```java
while (true) {
    WatchKey key = watchService.take();

    for (WatchEvent<?> event : key.pollEvents()) {
        WatchEvent.Kind<?> kind = event.kind();

        if (kind == StandardWatchEventKinds.OVERFLOW) {
            System.out.println("Se perdieron eventos");
            continue;
        }

        WatchEvent<Path> ev = (WatchEvent<Path>) event;
        Path nombreArchivo = ev.context();
        int count = ev.count();

        System.out.printf("Evento: %s, Archivo: %s, Count: %d%n",
            kind.name(), nombreArchivo, count);
    }

    boolean valido = key.reset();
    if (!valido) {
        break; // El directorio ya no es accesible
    }
}
```

Cada `WatchEvent` contiene:
- `kind()` -- el tipo de evento.
- `context()` -- el nombre del archivo relativo al directorio vigilado (como `Path`).
- `count()` -- el numero de veces que ocurrio el mismo evento (puede ser mayor que 1 si el sistema operativo acumulo eventos iguales).

Despues de procesar los eventos, es obligatorio llamar a `key.reset()` para volver a colocar la clave en estado de espera. Si `reset()` devuelve `false`, el directorio ya no esta accesible (fue eliminado o el WatchService se cerro) y debe dejarse de monitorear.

## Ejemplo completo

El siguiente programa monitorea un directorio y muestra en consola cualquier cambio que ocurra:

```java
public class DirectoryWatcher {
    public static void main(String[] args) throws IOException, InterruptedException {
        Path dir = Path.of(args.length > 0 ? args[0] : ".");
        WatchService ws = FileSystems.getDefault().newWatchService();
        dir.register(ws, ENTRY_CREATE, ENTRY_DELETE, ENTRY_MODIFY);

        System.out.println("Vigilando: " + dir.toAbsolutePath());

        while (true) {
            WatchKey key = ws.take();
            for (WatchEvent<?> event : key.pollEvents()) {
                WatchEvent.Kind<?> kind = event.kind();
                if (kind == OVERFLOW) continue;

                WatchEvent<Path> ev = (WatchEvent<Path>) event;
                Path filename = ev.context();
                System.out.printf("[%s] %s%n", kind.name(), filename);
            }
            if (!key.reset()) break;
        }

        ws.close();
    }
}
```

## Limitaciones y consideraciones

`WatchService` no es recursivo de forma nativa. Solo vigila el directorio registrado, no sus subdirectorios. Para monitorear un arbol completo, debe recorrerse el arbol con `Files.walk()` y registrar cada subdirectorio individualmente.

Los eventos no incluyen informacion sobre el estado anterior ni el estado posterior del archivo. Por ejemplo, para un evento `ENTRY_MODIFY`, no se sabe que parte del archivo cambio. Si la aplicacion necesita conocer el contenido modificado, debe leer el archivo completo tras recibir la notificacion.

En Linux, `inotify` tiene limitaciones de recursos: el numero de watches disponibles esta limitado por los parametros del kernel `fs.inotify.max_user_watches` y `fs.inotify.max_user_instances`. En sistemas con muchos archivos vigilados, puede ser necesario aumentar estos limites.

> A pesar de estas limitaciones, `WatchService` elimina la necesidad del polling y proporciona una integracion directa con los mecanismos de notificacion del sistema operativo, lo que lo hace significativamente mas eficiente que cualquier solucion basada en sondeo periodico.
