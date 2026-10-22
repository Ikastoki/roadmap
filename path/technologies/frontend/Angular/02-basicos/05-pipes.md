<h1 align="center">Pipes</h1>

<h2>📑 Contenido</h2>

- [Pipes](#pipes)
- [Pipes predefinidos](#pipes-predefinidos)
- [Pipes propios (personalizados)](#pipes-propios-personalizados)
  - [¿Cómo crear uno?](#cómo-crear-uno)

## Pipes

Los Pipes son una herramienta en Angular que te permite transformar datos en tus plantillas (templates) de una manera sencilla y declarativa. Piensa en ellos como filtros de datos: toman una entrada y devuelven un resultado transformado, sin modificar la fuente de datos original del componente.

El uso de un pipe se indica con el símbolo de barra vertical (|).

## Pipes predefinidos

Angular ofrece una gran cantidad de pipes listos para usar que cubren las transformaciones más comunes. Aquí tienes algunos de los más utilizados:

- `DatePipe`: Formatea una fecha según las reglas del locale (idioma y región).

  ```html
  <p>Fecha de hoy: {{ fechaActual | date:'dd/MM/yyyy' }}</p>
  ```

- `UppercasePipe` y `LowercasePipe`: Convierten una cadena de texto a mayúsculas o minúsculas, respectivamente.

  ```html
  <p>Nombre en mayúsculas: {{ 'angular' | uppercase }}</p>
  ```

- `CurrencyPipe`: Formatea un número como moneda.

  ```html
  <p>Precio: {{ 1250.75 | currency:'EUR':'symbol' }}</p>
  ```

- `DecimalPipe`: Formatea un número decimal con un número específico de dígitos.

  ```html
  <p>Puntuación: {{ 3.14159 | number:'1.2-2' }}</p>
  ```

- `JsonPipe`: Convierte un objeto o array a su representación en formato JSON, ideal para depuración.

  ```html
  <pre>{{ usuario | json }}</pre>
  ```

## Pipes propios (personalizados)

Si necesitas una transformación de datos que no existe en los pipes predefinidos, puedes crear tu propio pipe.

### ¿Cómo crear uno?

Al igual que con los componentes y servicios, el Angular CLI facilita la creación de pipes con el siguiente comando:

```bash
ng generate pipe nombre-de-tu-pipe
# O la versión corta
ng g p mi-pipe-personalizado
```

Esto generará un archivo `mi-pipe-personalizado.pipe.ts` con la siguiente estructura básica:

```typescript
import { Pipe, PipeTransform } from "@angular/core";

@Pipe({
  name: "miPipePersonalizado",
})
export class MiPipePersonalizadoPipe implements PipeTransform {
  transform(value: any, ...args: any[]): any {
    // Aquí va tu lógica de transformación
    return value;
  }
}
```

- `@Pipe`: El decorador que define el pipe y le asigna un nombre (name) para usarlo en las plantillas.

- `PipeTransform`: La interfaz que debes implementar para que Angular sepa que es un pipe.

- `transform()`: El método principal donde colocas la lógica de transformación. Recibe el valor de entrada (value) y los parámetros adicionales (args).
