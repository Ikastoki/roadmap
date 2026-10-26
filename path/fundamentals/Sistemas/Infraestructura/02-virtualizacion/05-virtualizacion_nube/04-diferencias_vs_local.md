# Diferencias clave vs. virtualizacion local

La virtualizacion en la nube y la virtualizacion local comparten los mismos principios tecnicos (hipervisor, VMs, aislamiento), pero difieren significativamente en como se consumen, gestionan y pagan los recursos. Estas diferencias determinan cuando es adecuado usar cada modelo.

## Modelo de costos

**Virtualizacion local (CAPEX):**
- Inversion inicial en hardware (servidores, almacenamiento, red).
- Costos fijos independientemente del uso.
- Gastos recurrentes: energia, refrigeracion, mantenimiento, espacio en rack.
- El hardware se amortiza en 3-5 anos.

**Nube (OPEX):**
- Sin inversion inicial.
- Pago por uso real (por hora o segundo).
- Elasticidad: pagas mas cuando escalas, menos cuando reduces.
- Riesgo de costos inesperados si no se monitorea.

```
Escenario: necesitas 32 GB de RAM para un proyecto de 6 meses

Local:
  Compra de servidor: 5000 EUR (unica vez)
  Energia + mantenimiento: 150 EUR/mes x 6 = 900 EUR
  Total: 5900 EUR

Nube:
  Instancia r5.xlarge (32 GB): 0.252 USD/hora
  Uso continuo 6 meses: 0.252 x 2160 horas = 544 USD (~500 EUR)
  Total: ~500 EUR

Pero si el proyecto dura 3 anos:
  Local: 5000 + 5400 = 10400 EUR
  Nube: 1800 EUR/año x 3 = 5400 EUR
  La nube sigue siendo mas barata
```

## Elasticidad

| Aspecto | Local | Nube |
|---|---|---|
| Escalar CPU | Comprar servidor nuevo | Cambiar tipo de instancia |
| Tiempo para escalar | Dias/semanas | Minutos |
| Reducir capacidad | Hardware ocioso sin reembolso | Dejar de pagar instancias |
| Escalado automatico | No posible | Configurable con auto-scaling |

## Control y visibilidad

**Local:** control total sobre hardware, red, almacenamiento y configuracion. Sabes exactamente que CPU tienes, que discos estan conectados y como esta cableada la red.

**Nube:** menor control sobre el hardware subyacente. No eliges el modelo de CPU exacto ni el modelo de disco. A cambio,获得了 escalabilidad, disponibilidad y servicios gestionados.

## Disponibilidad y recuperacion

**Local:** la alta disponibilidad requiere hardware adicional (cluster, almacenamiento compartido, redundancia de red). El coste de HA es significativo.

**Nube:** la disponibilidad esta integrada en la plataforma. Balanceadores de carga, replicas automaticas, multiples zonas de disponibilidad. La HA es mas accesible pero tiene un coste adicional.

## Seguridad

**Local:** tu controlas toda la pila de seguridad: firewalls, segmentacion de red, fisica del datacenter, acceso fisico.

**Nube:** la seguridad es responsabilidad compartida. El proveedor protege la infraestructura fisica; tu proteges los datos, las configuraciones y el acceso. IAM (Identity and Access Management) reemplaza a las claves SSH estaticas.

## Portabilidad y lock-in

**Local:** las VMs son archivos que puedes mover a cualquier lugar. Usas estandares abiertos (OVF, QCOW2).

**Nube:** las serviciospecificos del proveedor (S3, Lambda, DynamoDB) crean dependencia. Migrar de AWS a Azure requiere reescribir servicios, no solo mover VMs.

## Cuando usar cada modelo

- **Nube:** proyectos temporales, startups, aplicaciones con trafico variable, prototipado rapido, entornos de desarrollo/pruebas.
- **Local:** datos sensibles con regulacion estricta, cargas de trabajo estables con uso constante, requisitos de latencia ultra-baja, presupuesto fijo predecible.
- **Hibrido:** datos criticos en local, picos de demanda en nube, migracion gradual.

> La decision entre local y nube no es binaria. Muchas organizaciones operan en un modelo hibrido: mantienen infraestructura local para cargas estables y sensibles, y usan la nube para elasticidad, disaster recovery y servicios gestionados. Comprender las diferencias permite tomar la decision correcta para cada carga de trabajo.

