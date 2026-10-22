<h1 align="center">Observables </h1>

<h2>📑 Contenido</h2>

- [Observables (RxJS): El Corazón Asincrónico de Angular](#observables-rxjs-el-corazón-asincrónico-de-angular)
- [¿Qué son los Observables?](#qué-son-los-observables)
- [Componentes Clave de un Observable](#componentes-clave-de-un-observable)
- [Suscribirse y Manejar Errores](#suscribirse-y-manejar-errores)
- [Operadores RxJS Comunes](#operadores-rxjs-comunes)

## Observables (RxJS): El Corazón Asincrónico de Angular

RxJS (Reactive Extensions for JavaScript) es una biblioteca para gestionar eventos asincrónicos y flujos de datos. Los Observables son la herramienta principal de RxJS y son fundamentales en Angular, especialmente para manejar peticiones HTTP, eventos del usuario y cualquier tipo de dato que cambie con el tiempo.

## ¿Qué son los Observables?

Imagina un Observable como un flujo o "stream" de datos que puede emitir múltiples valores a lo largo del tiempo. A diferencia de una Promise que resuelve un único valor, un Observable puede emitir 0, 1 o muchos valores.

Una analogía sencilla es la de una suscripción a un periódico 📰.

- Observable: La empresa de periódicos, que produce y emite ejemplares (datos).

- Observer (o suscriptor): Tú, la persona que recibe los periódicos.

- Suscripción: La relación entre la empresa y tú. Sin una suscripción, la empresa no te enviará ningún periódico.

Una característica clave es que los Observables son perezosos. No hacen nada (no emiten valores) hasta que alguien se suscribe a ellos.

## Componentes Clave de un Observable

- Observable: La fuente de datos. Es una función que, al ser ejecutada, emite valores.

- Observer: Un objeto que define los callbacks para recibir los valores. Un Observer tiene tres métodos principales:

  - `next()`: Se llama cada vez que el Observable emite un nuevo valor.

  - `error()`: Se llama si el Observable encuentra un error.

  - `complete()`: Se llama cuando el Observable ha terminado de emitir valores.

- Subscription: El resultado de llamar al método `subscribe()`. Representa la ejecución en curso del Observable. Contiene el método `unsubscribe()` que te permite detener el flujo de datos y liberar recursos, lo cual es crucial para evitar pérdidas de memoria.

## Suscribirse y Manejar Errores

Para obtener los valores de un Observable, debes suscribirte a él. Aquí tienes un ejemplo completo, con el manejo de los tres callbacks:

```typescript
import { Component, OnInit, OnDestroy } from '@angular/core';
import { Observable, Subscription } from 'rxjs';

@Component({ ... })
export class DataComponent implements OnInit, OnDestroy {
  miSuscripcion: Subscription;
  datos: string[] = [];

  constructor() { }

  ngOnInit(): void {
    const miObservable$ = new Observable(observer => {
      // Emite tres valores
      observer.next('Valor 1');
      observer.next('Valor 2');
      // Puedes simular un error o completar la emisión
      // observer.error('¡Ha ocurrido un error!');
      observer.next('Valor 3');
      observer.complete();
    });

    // Suscribirse al Observable
    this.miSuscripcion = miObservable$.subscribe({
      next: (valor) => {
        console.log('Recibido:', valor);
        this.datos.push(valor);
      },
      error: (error) => console.error('Error:', error),
      complete: () => console.log('El flujo de datos ha terminado.')
    });
  }

  // Es crucial desuscribirse para evitar fugas de memoria
  ngOnDestroy(): void {
    this.miSuscripcion.unsubscribe();
  }
}
```

## Operadores RxJS Comunes

Los operadores son funciones que te permiten manipular y transformar el flujo de datos de un Observable. Se utilizan dentro del método `pipe()` del Observable.

| Operador    | Descripción                                                                | Ejemplo                                                                                                |
| ----------- | -------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------ |
| `map`       | Transforma cada valor emitido en otro valor diferente.                     | Transforma un objeto `post` en solo su `title`.                                                        |
| `filter`    | Emite solo los valores que cumplen con una condición.                      | Filtra una lista de `posts` para mostrar solo los que tienen `userId === 1`.                           |
| `tap`       | Realiza una acción secundaria sin modificar el valor del flujo.            | Se usa para hacer un `console.log()` o almacenar un valor sin afectar la cadena de operadores.         |
| `switchMap` | Un operador avanzado que cancela una petición anterior si llega una nueva. | Perfecto para barras de búsqueda: si escribes a, b, c, d rápidamente, solo se hará la petición para d. |

Ejemplo de uso de operadores con HttpClient:

```typescript
import { Component, OnInit } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { map, filter } from 'rxjs/operators';

@Component({ ... })
export class DataComponent implements OnInit {
  postsFiltrados: any[] = [];
  private apiUrl = 'https://jsonplaceholder.typicode.com/posts';

  constructor(private http: HttpClient) { }

  ngOnInit(): void {
    this.http.get<any[]>(this.apiUrl)
      .pipe(
        // 1. Filtra los posts para obtener solo los del userId 1
        filter(posts => posts.filter(post => post.userId === 1)),
        // 2. Mapea el array filtrado para solo obtener el título
        map(posts => posts.map(post => post.title))
      )
      .subscribe(titulos => {
        console.log('Títulos de posts del usuario 1:', titulos);
      });
  }
}

```
