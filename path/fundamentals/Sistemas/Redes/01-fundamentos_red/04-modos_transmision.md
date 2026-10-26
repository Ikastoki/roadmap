# Modos de transmision

Los modos de transmision definen la direccion y el flujo de datos en un canal de comunicacion. Determinan si los datos pueden viajar en una sola direccion, en ambas direcciones simultaneamente o en ambas direcciones pero de forma alternada. La eleccion del modo de transmision afecta la eficiencia, la complejidad del hardware y los requisitos de ancho de banda de la comunicacion.

Cada modo de transmision tiene un compromiso entre simplicidad, capacidad y coste. El modo adecuado depende de la naturaleza de la comunicacion: si el flujo de datos es predominantemente unidireccional (como television), bidireccional asimetrico (como navegacion web) o bidireccional simetrico (como videollamadas).

## Simplex

En el modo **simplex**, la comunicacion es unidireccional: los datos fluyen solo en una direccion, del emisor al receptor. El receptor no puede enviar datos al emisor a traves del mismo canal.

```
Emisor ──────→ Receptor
         datos
```

Ejemplos de simplex:

- Television y radio: la estacion transmite y el receptor solo escucha.
- Teclado a computadora: el teclado envia pulsaciones pero no recibe datos del PC por el mismo canal (aunque puede tener retroalimentacion LED por un canal separado).
- Monitor a computadora: el monitor recibe datos de video pero no envia datos al PC por el mismo cable.

El simplex es el modo mas simple y eficiente para comunicaciones donde no se necesita interaccion. El canal completo se dedica a una sola direccion, maximizando el ancho de banda util.

## Half-duplex

En el modo **half-duplex**, la comunicacion es bidireccional pero los datos solo pueden fluir en una direccion a la vez. Los dispositivos pueden enviar y recibir, pero no simultaneamente: cuando uno envia, el otro debe esperar.

```
Emisor ←─────→ Receptor
         datos (alternados)
```

Ejemplos de half-duplex:

- Walkie-talkie: solo un usuario puede hablar a la vez; el otro escucha. Cuando el primero suelta el boton, el otro puede responder.
- Ethernet clasico (10BASE2 con bus): solo un dispositivo puede transmitir en el bus en un momento dado.
- Comunicaciones por radio bidireccionales.

El half-duplex es mas eficiente que el simplex porque permite la interaccion, pero es menos eficiente que el full-duplex porque el canal se desaprovecha la mitad del tiempo. Requiere mecanismos de control para evitar colisiones cuando ambos dispositivos intentan transmitir simultaneamente.

## Full-duplex

En el modo **full-duplex**, la comunicacion es bidireccional y simultanea: ambos dispositivos pueden enviar y recibir datos al mismo tiempo en el mismo canal.

```
Emisor ←─────→ Receptor
      datos simultaneos
```

Ejemplos de full-duplex:

- Telefonia moderna: ambas partes pueden hablar y escuchar al mismo tiempo.
- Ethernet moderno (100BASE-TX, 1000BASE-T): cada par de cables dedica un par a cada direccion, permitiendo transmision simultanea.
- Wi-Fi: aunque comparte el medio, los protocolos modernos permiten comunicacion bidireccional casi simultanea.
- Comunicaciones seriales RS-422/RS-485.

El full-duplex duplica eficazmente la capacidad del canal respecto al half-duplex, ya que ambos flujos operan simultaneamente. Requiere dos canales separados (o dos pares de cables) para cada direccion.

## Comparacion de modos

| Modo | Direccion | Simultaneidad | Ejemplo | Complejidad |
|---|---|---|---|---|
| Simplex | Unidireccional | N/A | Television, radio | Baja |
| Half-duplex | Bidireccional | Alternada | Walkie-talkie | Media |
| Full-duplex | Bidireccional | Simultanea | Telefonia, Ethernet moderno | Alta |

> La eleccion del modo de transmision esta determinada por la naturaleza de la aplicacion. Las aplicaciones de streaming de video (Netflix, YouTube) son fundamentalmente simplex: el servidor envia y el cliente recibe. Las videollamadas requieren full-duplex porque ambas partes envian y reciben audio/video simultaneamente. Las redes LAN modernas operan en full-duplex con Ethernet, eliminando las colisiones que eran un problema en las redes de bus con half-duplex.

