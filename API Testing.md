
------
# Definición

>Una **API (Application Programming Interface)** es un conjunto de reglas que permite que diferentes aplicaciones se comuniquen entre sí. En el contexto web, una API es como un intermediario entre un cliente (por ejemplo, una web o app) y un servidor.

>El API Testing es el proceso de comprobar que una API funciona correctamente. En lugar de probar la interfaz gráfica (botones, páginas), aquí pruebas directamente las peticiones y respuestas.

## ¿Qué se verifica en API Testing?

✅ Que las respuestas sean correctas
✅ Que los códigos HTTP sean adecuados (200, 404, 500…)
✅ Que los datos tengan el formato esperado
✅ Que la API maneje bien errores
✅ Autenticación y permisos
✅ Rendimiento (tiempo de respuesta)

## Herramientas para probar APIs

- *Postman* - La que utilizo.
- *Insomnia*.

-------
# API recon

Empezar identificando los endpoints de la API.

``` http
GET /api/books HTTP/1.1
Host: example.com
```

En esta petición, se identifica el endpoint de la API `/api/books`. Otro endpoint posible podría ser `/api/books/mystery`.

Una vez identificados, determinar cómo interactuar con ellos. Para ello, hay que encontrar información sobre:

- *Los datos de entrada que procesa la API*, incluyendo parámetros obligatorios y opcionales.
- *Los tipos de petición que acepta la API*, incluyendo métodos HTTP soportados y formatos media.
- *Rate limits* y *mecanismos de autenticación*.

--------
# API documentation

## Discovering API documentation

Incluso cuando la documentación de la API no está disponible abiertamente, puede ser posible acceder a ella buscando en  aplicaciones que utilizan esta API.

Se puede hacer manualmente o con Burp Scanner. Buscar, por ejemplo:

- `/api`
- `/swagger/index.html`
- `/openapi.json`

Si se identifica un endpoint para un recurso de la API, investigar la ruta base. Por ejemplo, si se encuentra el endpoint `/api/swagger/v1/users/123`, probar:

- `/api/swagger/v1`
- `/api/swagger`
- `/api`

También se puede probar con un ataque de diccionario.

## Using machine-readable documentation

Utilizar herramientas automatizadas como Burp Scanner para analizar documentación en OpenAPI, o en JSON o YAML.

También se puede utilizar la extensión OpenAPI Parser.

Además, es posible utilizar herramientas especializadas para probar endpoints, como Postman o SoapUI.

----------
# Identifying API endpoints

Se puede conseguir mucha información buscando aplicaciones que utilizan la API, incluso cuando se tiene acceso a la documentación, porque esta puede ser incompleta o desactualizada.

Se busca por patrones como `/api`, y referencias dentro de archivos JavaScript.

Se pueden revisar los archivos `.js` con la extensión de BurpSuite *JS Link Finder*.

## Interacting with API endpoints

### Identifying supported HTTP methods

El endpoint puede soportar diferentes métodos HTTP, en ocasiones para distintas acciones.

Por ejemplo, el endpoint `/api/tasks` puede soportar:

- `GET /api/tasks` - Devuelve una lista de tareas.
- `POST /api/tasks` - Crea una nueva tarea.
- `DELETE /api/tasks/1` - Borra una tarea.

### Identifying supported content types

Los endpoints de la API suelen esperar un cierto formato para los datos, por lo que pueden comportarse diferente dependiendo del content type de los datos de la petición.

Puede permitir:

- *Provocar errores que filtren información útil*.
- *Evadir defensas*.
- *Aprovechar la diferencia del procesamiento lógico*. Por ejemplo, la API puede ser segura tratando JSON pero susceptible a ataques de inyección tratando con XML.

Se puede utilizar la extensión *Content type converter*.

### Brute-forcing to find hidden endpoints

Una vez identificado algún endpoint inicial, como:

`PUT /api/user/update`

Para identificar endpoints ocultos, se puede hacer fuerza bruta con otros recursos con la misma estructura. Por ejemplo, se puede probar a sustituir `/update` por otras funciones comunes, como `delete` y `add`.

Utilizar diccionarios que se basen en convenciones de nombres comunes de API y términos de la industria. También incluir términos relevantes para la aplicación.

-------
# Finding hidden parameters

Herramientas para identificar paŕametros ocultos:

- *Burp Intruder* - Usando el diccionario de nombres de parámetro comunes, que remplacen parámetros existentes o añadan nuevos.
- *Param miner* - Adivina nombres relevantes para la aplicación automáticamente, basándose en información tomada del scope.
- *Content discovery* - Descubre contenido que no está vinculado con contenido visible, que puedes buscar, incluyendo parámetros.

## Mass assignment vulnerabilities

Permite crear parámetros ocultos, que nunca fueron pensados para ser procesados por el desarrollador.

### Identifying hidden parameters

Identificar los parámetros ocultos examinando objetos devueltos por la API.

Por ejemplo, considerando la petición `PATCH /api/users/`, que permite a los ussuarios actualizar su nombre de usuario y email, e incluye el siguiente JSON:

``` json
{
	"username": "wiener",
	"email": "wiener@example.com"
}
```

Una petición concurrente a `GET /api/users/123` devuelve el siguiente JSON:

``` json
{
	"id": 123,
	"name": "John Doe",
	"email": "john@example.com",
	"isAdmin": false
}
```

Esto indica existen los parámetros ocultos `id` y `isAdmin` en el objeto interno del usuario, a parte de los parámetros actualizables username y email.

### Testing mass assignment vulnerabilities

Para probar si se puede modificar el valor del parámetro `isAdmin`, se añade a la petición `PATCH`:

``` json
{
	"username": "wiener",
	"email": "wiener@example.com",
	"isAdmin": false
}
```

Además, introducir un valor inválido para `isAdmin`:

``` json
{
	"username": "wiener",
	"email": "wiener@example.com",
	"isAdmin": "foo"
}
```

Si la aplicación se comporta diferente, quiere decir que el valor inválido afecta la consulta lógica, pero el valor válido no.

Para explotarlo, se puede mandar un valor `true` en el parámetro `isAdmin`:

``` json
{
	"username": "wiener",
	"email": "wiener@example.com",
	"isAdmin": true
}
```

Consiguiendo escalada de privilegios.

---------
# Server-side parameter pollution

## Testing for server-side parameter pollution in the query string

Probar caracteres de sintaxis de la query como `#`, `&` y `=` y observar como responde la aplicación.

Considerando una aplicación vulnerable que permite buscar otros usuarios basándose en el username, el navegador realiza la siguiente petición:

``` url
GET /userSearch?name=peter&back=/home
```

El servidor realiza la siguiente petición a la API interna:

``` url
GET /users/search?name=peter&publicProfile=true
```

### Truncating query strings

SE puede tratar de truncar la query string con el caracter `#` url-encodeado.

``` url
GET /userSearch?name=peter%23foo&back=/home
```

``` url
GET /users/search?name=peter#foo&publicProfile=true
```

Si la aplicación devuelve el usuario `peter`, la query ha sido truncada. Si por lo contrario devuelve un error de `Invalid name`, puede que se haya tratado `foo` como parte del username.

Si se es capaz de truncar la query string, se elimina el requerimiento del campo `publicProfile`.

### Injecting invalid parameters

Inyectar parámetros existentes (encontrados mediante uno de los métodos para encontrar parámetros ocultos), y ver si parseados y producen cambios en la respuesta.

``` url
GET /userSearch?name=peter%26email=foo&back=/home
```

``` url
GET /users/search?name=peter&email=foo&publicProfile=true
```

### Overriding existing parameters

Inyectar un segundo parámetro con el mismo nombre:

``` url
GET /userSearch?name=peter%26name=carlos&back=/home
```

``` url
GET /users/search?name=peter&name=carlos&publicProfile=true
```

El impacto depende de cómo procesa la aplicación el segundo parámetro:

- *PHP* parsea solo el último parámetro. Resulta en la búsqueda de `carlos`.
- *ASP.NET* combina ambos parámetros. Resulta en la búsqueda de `peter,carlos`, que puede devolver `Invalid username`.
- *Node.js* / *express* parsea solo el primer parámetro. Resulta en la búsqueda de `peter`.

## Testing for server-side parameter pollution in REST paths

Una API RESTful puede colocar los nombres de parámetros y sus valores en el path URL, en vez de en la query string:

``` url
/api/users/123
```

El path URL se tratará así:

- `/api` - Endpoint raíz de la API.
- `/users` - Representa un recurso (`users`).
- `/123` - Representa un parámetro.

Considerando una aplicación que permite editar los perfiles de usuario basándose en el username:

``` url
GET /edit_profile.php?name=peter
```

``` url
GET /api/private/users/peter
```

El atacante podría explotarlo mediante path traversal:

``` url
GET /edit_profile.php?name=peter%2f..%2fadmin
```

``` url
GET /api/private/users/peter/../admin
```

Si el cliente del lado del servidor o el back-end de la API normalizan el path, puede resolver a `/api/private/users/admin`.

## Testing for server-side parameter pollution in structured data formats

Cuando los datos se incluyen en estructuras como JSON o XML.

*El primer caso*:

``` http
POST /myaccount
name=peter
```

``` http
PATCH /users/7312/update
{"name":"peter"}
```

Se puede intentar añadir un parámetro `access_level`:

``` http
POST /myaccount
name=peter","access_level":"administrator
```

``` http
PATCH /users/7312/update
{"name":"peter","access_level":"administrator"}
```

*Segundo caso*:

``` http
POST /myaccount
{"name":"peter"}
```

``` http
PATCH /users/7312/update
{"name":"peter"}
```

``` http
POST /myaccount
{"name":"peter\",\"access_level\":\"administrator"}
```

Si el input se decodifica:

``` http
PATCH /users/7312/update
{"name":"peter","access_level":"administrator"}
```