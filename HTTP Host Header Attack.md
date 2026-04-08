--------------
# Definición

>Un **HTTP Host Header Attack** es un tipo de vulnerabilidad en la que un atacante manipula el encabezado `Host` de una petición HTTP para engañar al servidor o a la aplicación sobre qué dominio se está solicitando.

El problema ocurre cuando la aplicación confía en el valor del encabezado `Host` sin validarlo, usándolo para construir enlaces, redirecciones, correos, generación de tokens o lógica interna. Al alterar ese valor, el atacante puede provocar comportamientos inesperados como:

- Envenenamiento de caché (*web cache poisoning*)

- Generación de *enlaces maliciosos*

- Bypass de *controles de seguridad*

- *Secuestro de sesiones* o restablecimientos de contraseña manipulados

En esencia, el ataque explota la **falta de validación del dominio** que el servidor cree estar sirviendo.

--------------
# Identify host header vulnerabilities

## Supply an arbitrary host header

Para identificar vulnerabilidades de la cabecera `Host`, se empieza por introducir un valor arbitrario y ver cómo reacciona el servidor.

En ocasiones, la petición llegará al servidor igualmente. Esto puede ser por diversos motivos, pero uno de ellos es que el servidor tenga configurada una opción por defecto para cuando se recibe un host no esperado.

Otras veces simplemente mostrará un error del tipo `Invalid Host header` porque el servidor front-end no es capaz de redirigir la petición al servidor back-end correspondiente. En este caso, es necesario probar con otras técnicas.

## Check for flawed validation

Si al introducir un host arbitrario el servidor devuelve un mensaje de error diferente de `Invalid Host header` puede indicar que se está bloqueando por obra de un sistema de seguridad implementado. Por ejemplo, el servidor puede que esté validando si el valor de host coincide con el Server Name Indication (SNI) del TLS handshake. Sin embargo, esto no quiere decir que sea inmune a Host header attacks.

Es importante entender cómo el servidor está parseando el Host de la petición. Algunos casos vulnerables debidos al parsing son:

1- El algoritmo omite el puerto de la cabecera Host, resultando en que solo el dominio se valida. Si también permite introducir valores no numéricos en el puerto, se puede dejar el dominio intacto y aprovechar este vector para introducir el payload.

``` http
GET /test HTTP/2
Host: target.com:bad-stuff-here
```

2- Puede que el algoritmo aplique reglas lógicas para aceptar subdominios arbitrarios. En este caso, puede ser posible saltarse la validación completamente registrando un nuevo dominio que termine con la misma secuencia de caracteres que un dominio permitido.

``` http
GET /test HTTP/2
Host: nottarget.com
```

También se puede probar a introducir un subdominio menos seguro que ya ha sido comprometido:

``` http
GET /test HTTP/2
Host: hacked-subdomain.target.com
```

## Send ambiguous requests

En ocasiones, el código que valida el host y el código que hace algo vulnerable con él residen en diferentes componentes de la aplicación, o incluso en servidores diferentes. Identificando y explotando discrepancias en cómo reciben la cabecera pueden permitir enviar una petición ambigua que parece tener diferente Host dependiendo de que sistema la vea.

### Injecting duplicate host headers

Como el navegador no permite enviar una petición con dos cabeceras `Host`, es probable que los desarrolladores no hayan anticipado este escenario.

Aprovecha discrepancias entre las tecnologías de la aplicación a la hora de decidir qué cabecera tiene preferencia.

``` http
GET /example HTTP/1.1
Host: vulnerable-website.com
Host: bad-stuff-here
```

En el caso de que el front-end se quede con la primera ocurrencia, y el back-end con la segunda, se podría enrutar la petición al objetivo intencionado a través de la segunda cabecera.

### Supply an absolute URL

Aunque la línea de petición normalmente especifica un path relativo, muchos servidores están configurados para entender peticiones con URLs absolutas.

La ambigüedad entre la URL absoluta y la cabecera `Host` puede llevar a discrepancias entre diferentes sistemas.

``` http
GET https://vulnerable-website.com/ HTTP/1.1
Host: bad-stuff-here
```

*\*Nota*: Puede que el servidor se comporte diferente dependiendo el protocolo que se indique en la URL absoluta (HTTP / HTTPS).

### Add line wrapping

Se puede tratar de ocultar una de las cabeceras `Host` añadiendo un espacio delante. Algunos servidores interpretarán la cabecera sangrada como una línea envuelta, que es parte de la cabecera anterior. Otros servidores directamente la ignoran.

Como hay tantas inconsistencias al tratarlo, es común encontrar discrepancias entre los diferentes sistemas.

``` http
GET /example HTTP/1.1
	Host: bad-stuff-here
Host: vulnerable-website.com
```

Esto puede servir en el caso de que el servidor bloquee cabeceras Host duplicadas.

### Other techniques

Hay otras posibles vías para construir peticiones ambiguas, que siguen procedimientos típicos de [[HTTP Request Smuggling]].

## Inject host override headers

En arquitecturas de servidores con balanceador de carga o proxy inverso, es común que el front-end inyecte la cabecera `X-Forwarded-Host` conteniendo el valor original de la cabecera `Host` de la petición inicial del cliente.

Es por eso que muchos frameworks se refieren a esta cabecera, cuando está presente, en vez de a la cabecera Host. Esto puede suceder incluso cuando el front-end no inyecta esta cabecera.

``` http
GET /example HTTP/1.1
Host: vulnerable-website.com
X-Forwarded-Host: bad-stuff-here
```

Otras variantes:

- `X-Host`
- `X-Forwarded-Server`
- `X-HTTP-Host-Override`
- `Forwarded`

----------
# Exploiting HTTP Host Header

## Password reset poisoning

La modificación de la cabecera `Host` puede cambiar el enlace de cambio de contraseña enviado por correo.

## Web cache poisoning

Si el contenido de la cabecera `Host` o sus variantes se refleja en la página, y esta también utiliza una caché web, se pueden utilizar en conjunto para lanzar típicamente ataques XSS a otros usuarios.

Las cachés web normalmente incluyen la cabecera Host en la clave caché, por lo que hay que buscar otras alternativas.

## Classic server-side vulnerabilities

La cabecera `Host` puede ser un vector para explotar vulnerabilidades clásicas del lado del servidor, como puede ser SQL Injection, si el contenido se pasa a una sentencia SQL.

## Accessing restricted functionality

Algunas páginas bloquean funcionalidad interna mediante la comprobación de la cabecera `Host`. Si se modifica se puede lograr acceder a ellas.

## Accessing internal websites with virtual host brute-force

Sucede en casos en los que las empresas hostean su página web pública, y sitios privados internos en el mismo servidor.

Acceder al servidor mediante la IP pública resultará en la interacción con la página pública, pero acceder a través de la IP privada puede resultar en la interacción con el servicio interno.

``` http
www.example.com: 12.34.56.78
intranet.example.com: 10.0.0.132
```

Si el servicio interno no tiene asignado un registro DNS público, hay que hacer fuerza bruta por Hosts.

## Routing-based SSRF

Sucede en arquitecturas de servidores con balanceador de carga o proxy inverso.

Se identifica introduciendo un dominio propio en la cabecera `Host`. Si se recibe una consulta DNS del servidor objetivo u otro sistema de la arquitectura, se confirma que es posible enrutar peticiones a través del balanceador de carga o proxy inverso.

Se puede explotar este comportamiento para acceder a sistemas de la red interna. Para ello, se puede hacer fuerza bruta por nombres de dominio posibles, probar direccioes IP filtradas por la aplicación, o hacer fuerza bruta en un rango de IPs privado, como puede ser `192.168.0.0/16`.

## Connection state attacks

En conexiones reusadas, algunos servidores HTTP asumen que todas las peticiones tienen la misma cabecera `Host`, por lo que solo validan la primera petición.

Entonces, se puede enviar una petición inocente, y después, sobre la misma conexión, enviar la petición maliciosa con la cabecera cambiada.

En el caso de proxies inversos, si asumen que todas las peticiones de la misma conexión son hacia el mismo componente back-end, se puede explotar con la misma técnica para conseguir routing-based SSRF, password reset poisoning, y cache poisoning.

## SSRF via a malformed request line

Sucede cuando los proxies no validan correctamente la request line.

Por ejemplo, un proxy inverso puede coger el path de la request line, y prefijarlo con `http://backend-server`, y enrutar la petición a esa dirección URL.

El problema de esto es cuando el path no comienza por `/`, sino por `@`.

``` http
GET @private-intranet/example HTTP/1.1
```

La URL resultante será `http://backend-server@private-intranet/example`, que la mayoría de librerías HTTP interpretan como una petición para acceder a `private-intranet` con el usuario `backend-server`.