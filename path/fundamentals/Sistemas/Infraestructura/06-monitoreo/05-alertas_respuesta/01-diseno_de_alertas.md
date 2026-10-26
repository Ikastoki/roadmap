# Diseno de Alertas

El monitoreo sin alertas es un tablero decorativo: muestra datos, pero nadie reacciona cuando algo falla. El diseno de alertas es la disciplria que transforma señales crudas de telemetria en decisiones accionables. Un sistema de alertas mal disenado genera mas dano que silencio: el ruido constante de falsas alarmas adormece a los equipos, que terminan ignorando incluso las alertas criticas. Este fenomeno, conocido como *alert fatigue*, es la causa principal de que incidentes graves pasen desapercibidos. Disenar alertas efectivas implica entender que no toda anomalia merece una notificacion: solo aquellas que requieren intervencion humana justifican interrumpir a alguien.

## Alertas basadas en sintomas frente a causas

La distincion mas critica en el diseno de alertas es la diferencia entre monitorizar causas y monitorizar sintomas. Una causa es lo que produce el problema (un disco lleno, una CPU saturada); un sintoma es lo que el usuario experimenta (tiempos de respuesta elevados, errores HTTP 500). Las alertas basadas en causas son tentadoras porque son directas: "CPU al 90%" suena preciso y util. Sin embargo, esa misma CPU al 90% puede ser perfectamente normal en un servidor de procesamiento por lotes, mientras que en un servidor web indica un problema real. La causalidad varia segun el contexto, y el sistema no siempre comprende ese contexto.

Las alertas basadas en sintomas, en cambio, capturan directamente el impacto al usuario. "La latencia de las peticiones excede 2 durante 5 minutos" es una afirmacion que no necesita interpretacion: si se cumple, hay un problema real que afecta a alguien. Esta aproximacion reduce dramaticamente las falsas alarmas porque se centra en el efecto observable, no en la causa subyacente que podria ser transitoria o irrelevante.

```
# Alerta basada en causa (problematica)
IF cpu_usage > 90% FOR 5m THEN ALERT

# Alerta basada en sintoma (preferible)
IF http_request_duration_p99 > 2s FOR 5m THEN ALERT
```

## Niveles de severidad

Cada alerta necesita una clasificacion de severidad que determine como y a quien se notifica. El modelo mas comun utiliza tres niveles, cada uno con un protocolo de respuesta distinto:

**Critica (P1):** Indica un servicio completamente caido o datos criticos perdidos. Requiere respuesta inmediata, incluso fuera de horario laboral. Se notifica por multiples canales simultaneos (telefono, SMS, aplicacion de mensajeria). Ejemplo: "El servicio de autenticacion no responde desde hace 3 minutos". Un equipo debe ser despertado si es necesario.

**Advertencia (P2):** Indica degradacion significativa pero no catastrofica. El servicio funciona pero con impacto medible en los usuarios. Se notifica por canales de mensajeria en horario laboral. Ejemplo: "El percentil 99 de latencia supera el umbral establecido". Requiere atencion pronta pero no urgente.

**Informativa (P3):** Cambios de estado, metricas fuera de rangos esperados pero sin impacto inmediato. Solo se registra en paneles de monitoreo, sin notificaciones activas. Ejemplo: "El consumo de memoria ha superado el 75%". Sirve para revisiones periodicas y planeacion de capacidad.

> El error mas comun es marcar todo como critica. Cuando todo es urgente, nada lo es. Un equipo que recibe 50 alertas criticas al dia simplemente deja de mirar el telefono.

## Uebrales dinamicos frente a estaticos

Los umbrales estaticos son lineas fijas en el codigo de alerta: "alertar si los errores superan el 5%". Son simples de implementar pero problematicos en la practica. Un servicio que normalmente tiene 0.01% de errores veria una alerta al 1%, que podria ser una campana de marketing exitosa en vez de un fallo tecnico. Por otro lado, un servicio que opera normalmente al 4% de errores no generaria alerta hasta el 5%, cuando el problema ya lleva tiempo activo.

Los umbrales dinamicos se adaptan al comportamiento historico del sistema. Utilizan estadisticas moviles, desviaciones estandar o tecnicas de *machine learning* para establecer lo que es "normal" en cada momento. Por ejemplo, si la latencia del percentil 99 oscila entre 200ms y 400ms durante el dia, un umbral dinamico podria disparar una alerta cuando se supera el doble de la media movil de las ultimas 4 horas. Esto permite detectar anomalias reales sin depender de que alguien adivine el valor "correcto" del umbral.

La implementacion de umbrales dinamicos requiere un historial suficiente de datos. En servicios nuevos o con patrones de tráfico impredecibles, los umbrales estaticos conservadores son mas practicos. Lo ideal es comenzar con estaticos basados en experiencia y migrar a dinamicos a medida que se acumula historial.

## Fatiga de alertas

La fatiga de alertas es el fenomeno en el que los equipos se vuelven insensibles a las notificaciones por exceso de ruido. Es un problema organico que se desarrolla gradualmente: al principio, cada alerta genera atencion; despues de meses de falsas alarmas, las alertas reales se mezclan con el ruido y pasan desapercibidas. Estudios de la industria estiman que entre el 30% y el 50% de las alertas en organizaciones maduras son innecesarias.

Las estrategias para combatir la fatiga incluyen:

- **Alertar solo sobre accion requerida.** Si la alerta no tiene un "runbook" claro que indique que hacer, probablemente no deberia existir. Una alerta informativa sin accion definida solo genera ansiedad sin resolver nada.
- **Consolidar alertas multiples.** Si un servidor con 10 microservicios falla, no se necesitan 10 alertas. Una sola alerta que indique "el host X esta caido" es suficiente, con los servicios afectados como contexto adicional.
- **Aplicar supresion inteligente.** Durante mantenimientos programados o cuando se conoce un problema temporal, las alertas deben suprimirse automaticamente para no saturar los canales.
- **Revisar y podar regularmente.** Las alertas deben auditarse periodicamente. Las que no se han disparado en meses, o las que siempre se resuelven ignorandolas, candidatas a eliminar.

> Una buena regla practice: si una alerta se resuelve con "acknowledge and ignore" mas de tres veces al mes, debe reevaluarse o eliminarse.

