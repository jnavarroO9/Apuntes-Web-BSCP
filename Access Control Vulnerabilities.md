
-----------
# Definición

>Una **Access Control Vulnerability** (vulnerabilidad de control de acceso) ocurre cuando una aplicación o sistema no aplica de forma correcta las políticas que limitan qué usuarios pueden acceder a qué recursos o realizar qué acciones.

En otras palabras, son fallos que permiten a un usuario **evadir restricciones** y obtener permisos que no debería tener, como:

- **Ver** información privada de otros usuarios.
    
- **Modificar**, eliminar o crear datos sin autorización.
    
- **Acceder** a funciones administrativas sin ser administrador.

Este tipo de vulnerabilidades suele aparecer por configuraciones incorrectas o una validación deficiente en el lado del servidor, y son muy críticas porque afectan directamente a la **confidencialidad, integridad y disponibilidad** de la aplicación.

--------
# Unprotected admin panel

## Location

- Puede encontrarse la referencia a la ruta del panel de administración en el archivo *robots.txt*.
- También puede encontrarse en el código fuente de un recurso de la web:
	``` html
	<script>
		var isAdmin = false;
		if (isAdmin) {
		   var topLinksTag = document.getElementsByClassName("top-links")[0];
		   var adminPanelTag = document.createElement('a');
		   adminPanelTag.setAttribute('href', '/admin-6sxgq5');
		   adminPanelTag.innerText = 'Admin panel';
		   topLinksTag.append(adminPanelTag);
		   var pTag = document.createElement('p');
		   pTag.innerText = '|';
		   topLinksTag.appendChild(pTag);
		}
	</script>
	```

--------
# User role

## Controlled by request parameter

En la petición enviada, se puede ver una cookie:

``` http
Cookie: Admin=false
```

Cambiando el valor de la cookie a `true`, se logra escalar el privilegio a administrador.

## Can be modified in user profile

El estado de administrador se identifica con un atributo del usuario ``roleid`` con valor:

- `roleid=2`

Se puede llegar a modificar a través de una petición de cambio de email, donde se envia un json con la siguiente estructura:

``` json
{
	"email":"test@test.com",
	"roleid":2
}
```

--------
# User ID

## Controlled by request parameter

El panel de información de usuario para el usuario ``wiener`` se encuentra en:

``` URL
http://victim.com/my-account?id=wiener
```

Cambiando el valor de ``id`` se puede ver información de otros usuarios:

``` URL
http://victim.com/my-account?id=carlos
```

## Unpredictable user IDs

Los IDs de usuario son GUIDs aleatorios, pero se leakean en los href de los nombres de los autores de los posts.

``` html
<p><span id=blog-author><a href='/blogs?userId=3a4e5c38-6fd2-454a-a3d7-eaf38c3e82cb'>carlos</a></span> | 28 August 2025</p>
```

Introduciendolo como ``id`` en /my-account se obtiene movimiento lateral:

``` URL
http://victim.com/my-account?id=3a4e5c38-6fd2-454a-a3d7-eaf38c3e82cb
```

## Data leakage in redirect

Cambiando el valor de ``id`` a carlos en /my-account, se redirige a /login. Sin embargo, si se intercepta la petición, la respuesta de *302 FOUND* devuelve el contenido de /my-account de carlos.

*\*Nota*: Para interactuar con la página sin que se aplique redirect se pueden enviar las peticiones al proxy de burpsuite y en este activar la opción de reemplazar 302 FOUND -> 200 OK.

----------
# Insecure Direct Object References (IDORs)

Cuando se puede referenciar a objetos que pertenecen a la sesión de otro usuario.

\*Ejemplo de lab de portswigger:

- Hay un live chat con una funcionalidad de descargar el historial de mensajes.
- El botón redirige a: `/download-transcript/2.txt`.
	-El número es incremental. En la siguiente petición redirige a: `/download-transcript/3.txt`
- Como empieza en la 2, se puede tratar de referenciar al primer historial descargado.
	-`/download-transcript/1.txt`
- En este archivo se encuentra el historial de carlos, donde menciona cuál es su contraseña.

----------
# URL-based Access Control bypass

Si la restricción a un recurso se realiza desde el front-end, pero el back-end soporta de alguna manera la cabecera `X-Origin-URL`, se puede bypassear la restricción realizando una petición a un recurso accesible pero con la cabecera mencionada setteada al recurso restringido.

``` http
GET / HTTP/2
Host: victim.com
X-Origin-URL: /admin
```

-----------
# Method-based Access Control bypass

Puede ser el caso que las restricciones a un recurso se apliquen solo para un método de petición. Entonces, si se cambia el método, se puede acceder al recurso. Por ejemplo, si al recurso no deja acceder con GET, es posible que con POST si que se consiga.

----------
# Multi-step process with no access control on one step

Es posible que una funcionalidad privilegiada que requiera varias peticiones al servidor no controle correctamente el acceso en todos sus pasos. De ser el caso, y dependiendo del contexto de la funcionalidad y de cómo esté montada, se puede aprovechar ese paso sin control de acceso para ejecutar la función privilegiada.

Como ejemplo, en una página web con funcionalidad de convertir a administrador, primero se realiza la petición de upgrade y después se envía la petición de confirmación. La primera tiene control de acceso y devuelve 'Unauthorized'. Sin embargo, la segunda petición permite efectuar la acción sin restricción alguna.

---------
# Referer-based Access Control bypass

Si la única restricción para ejecutar una acción privilegiada es que se haga la petición desde el panel de administrador, y para ello se hace la comprobación desde el ``Referer``, puede bypassearse simplemente colocando la dirección URL del panel de administrador en la cabecera ``Referer`` de la petición maliciosa.
