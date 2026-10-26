# Serverless y funciones (FaaS)

Serverless (sin servidor) es el siguiente nivel de abstraccion despues de los contenedores. En el modelo serverless, el desarrollador no gestiona ni servidores, ni VMs, ni contenedores: solo escribe funciones que se ejecutan en respuesta a eventos. El proveedor de nube se encarga de todo lo demas: infraestructura, escalado, mantenimiento, disponibilidad.

FaaS (Function as a Service) es la implementacion tecnica de serverless: funcion es desplegada individualmente y se ejecuta cuando se produce un evento (peticion HTTP, mensaje en una cola, subida de un archivo, cambio en una base de datos).

## Arquitectura

```
Evento (HTTP, cola, archivo)
    │
    ▼
Proveedor de nube (AWS Lambda, Azure Functions, GCP Cloud Functions)
    │
    │  Escala automaticamente a cero cuando no hay trafico
    │  Escala automaticamente a miles de instancias con trafico
    │
    ▼
Funcion ejecutada
    │
    ▼
Respuesta (HTTP, escritura en BD, envio de email)
```

**No hay:**
- Servidor que configurar ni mantener.
- SO que parchear.
- Hipervisor que gestionar.
- Contenedor que orquestar.
- Balanceo de carga que configurar.
- Autoscaling que definir.

## Ejemplo basico

```javascript
// AWS Lambda: funcion que responde a una peticion HTTP
exports.handler = async (event) => {
    const nombre = event.queryStringParameters?.nombre || 'Mundo';

    return {
        statusCode: 200,
        body: JSON.stringify({
            mensaje: `Hola, ${nombre}!`
        })
    };
};
```

```python
# Python: funcion Lambda que procesa un archivo subido a S3
import boto3

def lambda_handler(event, context):
    bucket = event['Records'][0]['s3']['bucket']['name']
    key = event['Records'][0]['s3']['object']['key']

    s3 = boto3.client('s3')
    contenido = s3.get_object(Bucket=bucket, Key=key)['Body'].read()

    # Procesar el archivo...
    return {'statusCode': 200, 'body': 'Archivo procesado'}
```

## Ventajas

- **Escalado a cero:** no pagas cuando no hay trafico.
- **Escalado automatico:** de 0 a miles de instancias en segundos.
- **Sin mantenimiento:** el proveedor gestiona todo.
- **Pago por ejecucion:** solo pagas por los milisegundos que tu codigo ejecuta.
- **Velocidad de despliegue:** subir una funcion y que este activa en segundos.

## Limitaciones

- **Cold starts:** la primera invocacion despues de un periodo de inactividad tiene latencia adicional (500ms - 3s).
- **Tiempo de ejecucion limitado:** la mayoria de proveedores limitan a 5-15 minutos por ejecucion.
- **Sin estado:** cada ejecucion es independiente; el estado se debe almacenar externamente.
- **Vendor lock-in:** las funciones estan intimamente ligadas al proveedor.
- **Depuracion compleja:** los logs estan distribuidos en multiples servicios.
- **No apto para todo:** cargas de trabajo persistentes, bases de datos, websockets son problemáticas.

## Comparacion con virtualizacion

| Aspecto | VM | Contenedor | Serverless |
|---|---|---|---|
| Gestion | Tu gestionas todo | Tu gestionas el SO | El proveedor gestiona todo |
| Escalado | Manual/auto | Manual/orquestado | Automatico a cero |
| Arranque | Minutos | Segundos | Milisegundos |
| Pago | Siempre (instancia encendida) | Siempre (instancia encendida) | Solo cuando ejecuta |
| Estado | Persistente | Persistente | Efimero |
| Complejidad operativa | Alta | Media | Baja |

> Serverless no elimina la virtualizacion: la oculta. Detras de cada funcion Lambda hay una VM, un hipervisor y un servidor fisico. La diferencia es que el usuario ya no ve ni gestiona nada de eso. Es la evolucion final de la abstraccion: de gestionar servidores a escribir funciones.

