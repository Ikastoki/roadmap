<h1 align="center">HttpClient </h1>

<h2>📑 Contenido</h2>

- [Uso de HttpClient para Peticiones HTTP](#uso-de-httpclient-para-peticiones-http)
- [Configuración Inicial](#configuración-inicial)
- [Creación de un Servicio para la API](#creación-de-un-servicio-para-la-api)
- [Métodos HTTP Principales](#métodos-http-principales)
  - [GET: Obtener datos](#get-obtener-datos)
  - [POST: Crear un nuevo recurso](#post-crear-un-nuevo-recurso)
  - [PUT: Actualizar un recurso existente](#put-actualizar-un-recurso-existente)
  - [DELETE: Eliminar un recurso](#delete-eliminar-un-recurso)

## Uso de HttpClient para Peticiones HTTP

HttpClient es el módulo incorporado de Angular que te permite comunicarte con servidores remotos para obtener, enviar, actualizar o eliminar datos. Es un servicio que se basa en Observables, lo que facilita el manejo de peticiones asíncronas.

## Configuración Inicial

Antes de poder usar `HttpClient`, necesitas importarlo en el módulo principal de tu aplicación (AppModule).

- Abre `src/app/app.module.ts`.

- Importa `HttpClientModule` desde `@angular/common/http`.

- Añade `HttpClientModule` al array de `imports`.

```typescript
import { NgModule } from "@angular/core";
import { BrowserModule } from "@angular/platform-browser";
import { HttpClientModule } from "@angular/common/http"; // Importa HttpClientModule
import { AppComponent } from "./app.component";

@NgModule({
  declarations: [AppComponent],
  imports: [
    BrowserModule,
    HttpClientModule, // Añade HttpClientModule aquí
  ],
  providers: [],
  bootstrap: [AppComponent],
})
export class AppModule {}
```

## Creación de un Servicio para la API

La mejor práctica es encapsular toda la lógica de las peticiones HTTP en un servicio. Esto mantiene tus componentes limpios y reutilizables. Puedes crear un servicio usando el CLI:

```bash
ng generate service api
```

A continuación, inyectaremos el HttpClient en este nuevo servicio.

src/app/api.service.ts

```typescript
import { Injectable } from "@angular/core";
import { HttpClient } from "@angular/common/http";
import { Observable } from "rxjs";

@Injectable({
  providedIn: "root",
})
export class ApiService {
  private apiUrl = "https://jsonplaceholder.typicode.com/posts";

  constructor(private http: HttpClient) {}

  // Aquí irán las funciones para cada tipo de petición
}
```

## Métodos HTTP Principales

Una vez que el servicio está configurado, puedes empezar a definir las funciones para las peticiones. Cada método de `HttpClient` retorna un `Observable`, por lo que necesitarás suscribirte para obtener la respuesta.

### GET: Obtener datos

Se utiliza para solicitar recursos de un servidor.

```typescript
// En api.service.ts
getPosts(): Observable<any[]> {
  // Retorna un Observable con el array de posts
  return this.http.get<any[]>(this.apiUrl);
}

// En el componente que usa el servicio
import { Component, OnInit } from '@angular/core';
import { ApiService } from './api.service';

@Component({ ... })
export class PostsComponent implements OnInit {
  posts: any[] = [];

  constructor(private apiService: ApiService) { }

  ngOnInit(): void {
    // Te suscribes al Observable para obtener los datos
    this.apiService.getPosts().subscribe(data => {
      this.posts = data;
      console.log('Datos obtenidos:', this.posts);
    });
  }
}
```

### POST: Crear un nuevo recurso

Se utiliza para enviar datos al servidor para crear un nuevo recurso.

```typescript
// En api.service.ts
createPost(post: any): Observable<any> {
  return this.http.post<any>(this.apiUrl, post);
}

// En el componente que usa el servicio
createNuevoPost(): void {
  const nuevoPost = { title: 'Mi nuevo post', body: 'Contenido del post.', userId: 1 };
  this.apiService.createPost(nuevoPost).subscribe(response => {
    console.log('Post creado:', response);
  });
}
```

### PUT: Actualizar un recurso existente

Se utiliza para actualizar completamente un recurso ya existente en el servidor.

```typescript
// En api.service.ts
updatePost(id: number, post: any): Observable<any> {
  const url = `${this.apiUrl}/${id}`;
  return this.http.put<any>(url, post);
}

// En el componente que usa el servicio
actualizarPost(): void {
  const postActualizado = { id: 1, title: 'Título actualizado', body: 'Nuevo contenido', userId: 1 };
  this.apiService.updatePost(1, postActualizado).subscribe(response => {
    console.log('Post actualizado:', response);
  });
}
```

### DELETE: Eliminar un recurso

Se utiliza para eliminar un recurso específico del servidor.

```typescript
// En api.service.ts
deletePost(id: number): Observable<any> {
  const url = `${this.apiUrl}/${id}`;
  return this.http.delete<any>(url);
}

// En el componente que usa el servicio
eliminarPost(): void {
  this.apiService.deletePost(1).subscribe(() => {
    console.log('Post eliminado correctamente');
  });
}

```
