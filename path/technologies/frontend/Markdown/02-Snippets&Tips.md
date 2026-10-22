<h1 align="center">Snippets y Tips</h1>

<h2>Contenido</h2>

- [Snippets](#snippets)
	- [Nuevo archivo para apuntes](#nuevo-archivo-para-apuntes)
	- [Insertar código](#insertar-código)
	- [Insertar Tabla](#insertar-tabla)
	- [Centrar Contenido](#centrar-contenido)
	- [Insertar Paginado](#insertar-paginado)
	- [Insertar Paginado-Ejercicios](#insertar-paginado-ejercicios)
	- [Insertar Nota(Para GitHub)](#insertar-notapara-github)
	- [Insertar Consejo(Para GitHub)](#insertar-consejopara-github)
	- [Insertar Anotación Importante(Para GitHub)](#insertar-anotación-importantepara-github)
	- [Insertar Aviso(Para GitHub)](#insertar-avisopara-github)
	- [Insertar Precaución(Para GitHub)](#insertar-precauciónpara-github)

## Snippets

Un snippet es un pequeño fragmento de código, texto o datos que se utiliza con frecuencia o se comparte entre proyectos. Estos fragmentos suelen ser reutilizables y están destinados a facilitar tareas comunes al proporcionar una forma rápida de insertar o utilizar ciertas porciones de código.

### Nuevo archivo para apuntes

```json
//Archivo JSON
"Archivo en blanco(Apuntes)": {
	 	"prefix": "md-base",
	 	"body": [
	 		"<h1 align=\"center\">Tema</h1>\n",
	 		"<h2>Contenido</h2>\n",
			"- [TituloSección](#TituloSección)\n",
			"## TituloSección"
	 	],
	 	"description": "Esqueleto básico de los apuntes."
	 },
```

---

### Insertar código

````json
//Archivo JSON
"Insertar Código": {
		"prefix": "md-code",
		"body": [
			"**Código**",
			"```$1",
		   "$2",
		   "```"
		],
		"description": "Insertar código."
	},
````

---

### Insertar Tabla

```json
//Archivo JSON
"Insertar Tabla": {
		"prefix": "md-table",
		"body": [
			"| Columna | Columna | Columna | Columna |\n",
			"| :-----: | ------- | ------- | ------- |\n",
		    "|  Fila   | Fila    | Fila    | Fila    |\n",
		    "|  Fila   | Fila    | Fila    | Fila    |\n",
		],
		"description": "Insertar una tabla básica."
	},
```

---

### Centrar Contenido

```json
//Archivo JSON
"Centrar Contenido": {
		"prefix": "md-center",
		"body": [
			"<div align=\"center\">",
			"$1",
		    "</div>",
		],
		"description": "Centrar Contenido."
	},
```

---

### Insertar Paginado

```json
//Archivo JSON
"Insertar Paginado": {
		"prefix": "md-pag",
		"body": [
			"<div align=\"center\">",
			"<a href="">⬅️ Anterior</a>",
		    "&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;",
			"<a href="">Siguiente ➡️</a>",
			"</div>",
		],
		"description": "Agregar Paginado."
	},
```

---

### Insertar Paginado-Ejercicios

```json
//Archivo JSON
"Insertar Paginado-Ejercicios": {
		"prefix": "md-ejer",
		"body": [
			"<div align=\"center\">",
			"<a href="">🧑‍💻 Ejercicios</a>",
			"</div>",
		],
		"description": "Agregar Paginado Ejercicios."
	},
```

---

### Insertar Nota(Para GitHub)

```json
//Archivo JSON
"Insertar Nota": {
		"prefix": "md-note",
		"body": [
			"> [!NOTE]",
			"> ",
			"> $1",
		],
		"description": "Insertar una nota(Para GitHub)."
	},
```

---

### Insertar Consejo(Para GitHub)

```json
//Archivo JSON
"Insertar Consejo": {
		"prefix": "md-tip",
		"body": [
			"> [!TIP]",
			"> ",
			"> $1",
		],
		"description": "Insertar un consejo(Para GitHub)."
	},
```

---

### Insertar Anotación Importante(Para GitHub)

```json
//Archivo JSON
"Insertar Anotación Importante": {
		"prefix": "md-important",
		"body": [
			"> [!IMPORTANT]\n",
			"> $1",
		],
		"description": "Insertar un consejo o anotación importante(Para GitHub)."
	},
```

---

### Insertar Aviso(Para GitHub)

```json
//Archivo JSON
"Insertar Aviso": {
		"prefix": "md-warning",
		"body": [
			"> [!WARNING]",
			"> ",
			"> $1",
		],
		"description": "Insertar un aviso(Para GitHub)."
	},
```

---

### Insertar Precaución(Para GitHub)

```json
//Archivo JSON
"Insertar Precaución": {
		"prefix": "md-caution",
		"body": [
			"> [!CAUTION]",
			"> ",
			"> $1",
		],
		"description": "Insertar un aviso de precaución(Para GitHub)."
	},
```
