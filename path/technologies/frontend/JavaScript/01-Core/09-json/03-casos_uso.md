<h1 align='center'>Casos de uso de JSON</h1>

<h2>📑 Contenido</h2>

- [Intercambio de datos](#intercambio-de-datos)
- [Almacenamiento de configuración](#almacenamiento-de-configuración)
- [Archivos de datos](#archivos-de-datos)

## Intercambio de datos

JSON es ideal para el intercambio de datos entre un servidor y un cliente. Los datos enviados desde un servidor a una página web generalmente están en formato JSON. Al enviar datos desde el navegador al servidor, también suele utilizarse JSON.

Ejemplo de una respuesta de una API RESTful en formato JSON:

```json
{
  "usuario": {
    "id": 123,
    "nombre": "María",
    "correo": "maria@example.com"
  },
  "estado": "activo"
}
```

## Almacenamiento de configuración

JSON también es muy útil para almacenar datos de configuración o preferencias de usuarios, tanto en aplicaciones web como en archivos externos. Muchos entornos y bibliotecas utilizan archivos `.json` para gestionar configuraciones.

```json
{
  "configuracion": {
    "tema": "oscuro",
    "notificaciones": true,
    "idioma": "es"
  }
}
```

## Archivos de datos

Los archivos `.json` son ampliamente utilizados para almacenar datos de manera estructurada. Los desarrolladores a menudo los utilizan en aplicaciones como bases de datos ligeras, donde no se requiere un servidor de base de datos completo

Ejemplo de archivo config.json:

```json
{
  "database": {
    "host": "localhost",
    "port": 3306,
    "user": "root",
    "password": "password123"
  }
}
```
