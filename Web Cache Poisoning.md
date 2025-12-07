
------------
# Definición

>**Web Cache Poisoning** (en español, _envenenamiento de caché web_) es un tipo de ataque en el que un atacante **manipula las respuestas almacenadas en la caché de un servidor web o un proxy** para que los usuarios posteriores reciban contenido malicioso o incorrecto.

## Explicación

Muchos servidores y proxies utilizan **caché web** para mejorar el rendimiento, guardando temporalmente copias de respuestas HTTP (como páginas HTML, imágenes o scripts).  
El atacante aprovecha **vulnerabilidades en la forma en que el servidor maneja las solicitudes HTTP** (por ejemplo, encabezados, parámetros o validaciones insuficientes) para **inyectar contenido falso o peligroso** en la versión almacenada en caché.

Cuando otros usuarios solicitan la misma página, el sistema les entrega **la respuesta envenenada** desde la caché, sin pasar por el servidor original.

## Ejemplo sencillo:

1. El servidor cachea la página `https://ejemplo.com/inicio`.
    
2. Un atacante envía una solicitud manipulada con un encabezado o parámetro malicioso.
    
3. El servidor genera y almacena en caché una respuesta contaminada (por ejemplo, un script malicioso en el HTML).
    
4. Otros usuarios que acceden a `https://ejemplo.com/inicio` reciben la versión maliciosa desde la caché.

## Consecuencias:

- Inyección de **JavaScript malicioso (XSS persistente)**.
    
- **Robo de cookies o tokens de sesión**.
    
- **Redirecciones a sitios falsos**.
    
- **Manipulación del contenido mostrado** al usuario.

## Mitigaciones:

- Validar y normalizar todas las cabeceras y parámetros de entrada.
    
- Asegurar que solo contenido confiable se almacene en la caché.
    
- Configurar reglas de caché estrictas (`Cache-Control`, `Vary`, etc.).
    
- Usar listas blancas de cabeceras aceptadas para el almacenamiento.

---------
# Metodología de explotación

Asumiendo que la aplicación utiliza una caché para almacenar respuestas de las peticiones de los clientes durante cierto periodo de tiempo.

## 0. Utilizar un cache-buster

Como se van a realizar pruebas que involucran almacenar peticiones en la caché, lo que puede provocar que se sirvan recursos no intencionados a otros clientes, es conveniente utilizar un cache-buster. Consiste en un identificador único que se va a incluir en un parámetro que se tome como clave.

``` http
GET /?cb=1234 HTTP/1.1
Host: victim.com
.
.
.
```

En este caso, el cache-buster es: `?cb=1234`.

## 1. Identificar unkeyed headers

La cache emplea la primera línea de la petición y ciertas cabeceras como claves para identificar a un mismo recurso almacenado en caché. Entonces, si un servidor recibe distintas peticiones con la misma clave, y tiene una respuesta almacenada en caché asociada a esa clave, va a devolver a todas esas peticiones el mismo recurso.

Para un atacante, resulta de gran valor identificar cuáles son esas cabeceras que no se incluyen en la clave. Para llevarlo a cabo, se puede realizar una comprobación manual o utilizar la extensión de BurpSuite **Param Miner**, que realiza fuerza bruta para encontrar esas cabeceras, empleando de manera implícita el cache-buster.

### Uso de Param Miner

1. Se selecciona la petición que se cachea.
2. Click derecho > `Extensions > Param Miner > Guess headers`.
3. El output se puede ver en `Extensions > Installed > Param Miner > Output`.

## 1.2 Identificar unkeyed query params

También se puede tratar de identificar parámetros de la URL que no se incluyen en la clave y que se puedan usar para construir una respuesta maliciosa.

## 2. Provocar una respuesta maliciosa

El siguiente paso es conseguir inyectar algún payload malicioso que se vaya a almacenar después en la caché. Puede ser cualquier tipo de payload que se vea reflejado en la página, como un reflected XSS.

## 3. Conseguir cachear la respuesta

Para que la respuesta se sirva al resto de usuarios, se tiene que provocar un fallo en la caché, es decir, que no haya ninguna respuesta guardada para esa clave. Además, la clave con la que se almacene tiene que ser la misma que la clave que generará la petición de las víctimas. Por tanto, es necesario elliminar el cache-buster.

---------
# Via unkeyed query string

Si la query string no se incluye en la clave, y se muestra en el html sin sanitizar, puede servir para construir una respuesta maliciosa.

``` http
GET /?cb="><script>alert(document.cookie)</script> HTTP/2
Host: vulnerable.com
```

``` http
HTTP/2 200 OK
...

...<script>alert(document.cookie)</script>...
```

Para estos casos, si se necesita un cache-buster para realizar pruebas sin afectar a los usuarios, se puede buscar una cabecera que sí se incluya en la clave.

-----------
# Parameter cloaking

Si se da el caso de que uno de los parámetros de la query string es vulnerable, pero se está incluyendo en la clave caché, se puede tratar de aprovechar alguna *discrepancia en el parsing* de parámetros entre la *caché* y el servidor *back-end* para encubrir el parámetro y que no se incluya en la clave.

\*Ejemplo:

Se tiene la siguiente petición:
``` http
GET /js/geolocate.js?callback=setLocateCookie HTTP/2
Host: vulnerable.com
```

Y la respuesta:
``` http
HTTP/2 200 OK
...

...setLocateCookie...
```

El valor de ``callback`` se incluye sin sanitizar, por lo que es vulnerable a XSS. Pero este se incluye en la clave, por lo que si se altera el valor, ya no coincidiría con la clave de la respuesta de otro usuario, luego no se servirá a ningún usuario víctima.

Por otro lado, el back-end acepta como delimitadores de parámetros, además de ``&``, el `;`. Sin embargo, el parser de la caché no. Aprovechando esto, y un parámetro que no se incluye en la clave, se puede construir la siguiente petición:
``` http
GET /js/geolocate.js?callback=setLocateCookie&utm_content=1;callback=alert(1) HTTP/2
Host: vulnerable.com
```

Esta petición generará la clave: `/js/geolocate.js?callback=setLocateCookie` (utm_content se ignora), que es el valor sin alterar, igual al de otro usuario.

Pero tiene una diferencia, y es que el back-end ha leído los siguientes parámetros:
- callback=setLocateCookie
- utm_content=1
- callback=alert(1)

Ante una coincidencia, el servidor toma como valor la última ocurrencia, sirviendo como respuesta:
``` http
HTTP/2 200 OK
...

...alert(1)...
```

Consiguiendo así envenenar la caché con código javascript malicioso.

## Via fat GET request

Si el servidor acepta peticiones GET con cuerpo, y el cuerpo no se incluye en la clave caché, se puede utilizar para modificar el valor del parámetro sin alterar la clave.
``` http
GET /js/geolocate.js?callback=setLocateCookie HTTP/2
Host: vulnerable.com

callback=alert(1)
```

``` http
HTTP/2 200 OK
...

...alert(1)...
```

-----------
# URL normalization

Los ataques de reflected XSS donde el payload se encuentra en la URL suelen ser inexplotables, debido a que el navegador realiza un URL-encode antes de mandar la petición, y el servidor no lo decodifica. Sin embargo, sí que es explotable con Web Cache Poisoning.

Algunas implementaciones de cachés normalizan los valores que se introducen en la clave. De modo que:
- `GET /test?param=<script>alert(1)</script>`
- `GET /test?param=%3cscript%3ealert%281%29%3c%2fscript%3e`

estas dos peticiones tienen la misma clave.

Así pues, si se envenena la caché con la primera petición, y se comparte la segunda a un usuario víctima, al acceder al enlace cargará la respuesta envenenada con el XSS.

------------
# Cache key injection

>Este vector de ataque se emplea cuando la entrada vulnerable se almacena en la clave caché.

Para construir la clave caché, normalmente se concatenan los componentes de la clave (keyed components) utilizando ciertos limitadores ($\$ o \_\_). Si la caché no escapa correctamente los delimitadores, permite construir otra petición con la misma clave caché que la vulnerable.

Por ejemplo, realizar la siguiente petición obtiene la respuesta con un XSS:
``` http
GET /path?param=123 HTTP/1.1
Origin: '-alert(1)-'__

HTTP/1.1 200 OK
X-Cache-Key: /path?param=123__Origin='-alert(1)-'__

<script>…'-alert(1)-'…</script>
```

Los delimitadores (\_\_), no se escapan correctamente, y permite construir la siguiente petición, que resulta en la misma clave caché que la primera: 
``` http
GET /path?param=123__Origin='-alert(1)-'__ HTTP/1.1

HTTP/1.1 200 OK
X-Cache-Key: /path?param=123__Origin='-alert(1)-'__
X-Cache: hit

<script>…'-alert(1)-'…</script>
```

Por lo que se ve, si se induce a la víctima a realizar la segunda petición, se servirá el contenido envenenado por la primera.

-----------
# Internal cache poisoning

Algunas páginas web implementan una caché interna en la aplicación, en conjunto con la caché externa ya conocida.

Como estas cachés están estrechamente relacionadas con la aplicación, pueden llegar a comportarse de manera inusual con respecto a las cachés externas de propósito general, lo que abre la puerta a explotaciones de envenenamiento de la caché interna.

En vez de cachear respuestas enteras, algunas de estas cachés descomponen la respuesta en múltiples fragmentos reutilizables y los cachean por separado.

Como estos fragmentos están pensados para ser reutilizables para múltiples peticiones diferentes, el concepto de clave caché se rompe. Cualquier respuesta que contenga un fragmento cacheado reutilizará ese mismo fragmento, aunque el resto de la respuesta sea diferente. Por tanto, si se consigue envenenar un fragmento que se sirve en un gran número de respuestas, un gran número de usuarios serán afectados, solo con una sola petición.

## How to identify internal caches

- Si la respuesta devuelve contenido de la petición en cuestión, además de contenido de alguna petición previa, es un indicador clave de que se está empleando una caché interna para cachear fragmentos de la respuesta.
- Además, si el contenido cacheado se muestra en distintas páginas.
- Si la caché se comporta de una manera muy inusual, la respuesta más lógica es que se trata de una caché interna.

Si la web implementa varios niveles de caché, puede resultar muy difícil comprender qué está pasando detrás, y entender cómo se comporta el sistema de caché.