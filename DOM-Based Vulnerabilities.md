
--------------

# Definición

>Una **vulnerabilidad basada en el DOM** ocurre cuando el código JavaScript del lado del cliente lee datos de una fuente controlada por el usuario (como `location`, `document.URL`, `document.referrer`, etc.) y los utiliza para modificar dinámicamente el DOM o ejecutar funciones sin validar esos datos, lo que puede llevar a ataques como **XSS (Cross-Site Scripting)**, **open redirects**, entre otros.

## Características clave

- Ocurre *del lado del cliente* (JavaScript en el navegador).
    
- No involucra necesariamente tráfico al servidor.
    
- Suele ser más difícil de detectar con herramientas automáticas, ya que requiere análisis del comportamiento del JavaScript.

------------
# DOM XSS

## Usando Web Messages

- Si la página tiene un **eventListener** sobre un mensaje web, y este lo vuelca directamente en la página web, se puede aprovechar para mandar un mensaje con *postMessage()* inyectando código javascript.
	Para ejecutarlo en el navegador de la víctima, es necesario abrir un `<iframe>`, donde se enviará el mensaje a través del atributo *'contentWindow'*.

	``` html
	<iframe width=600px height=600px src="https://0ae300b304dbb07981b2c02600b400e1.web-security-academy.net/" onload='this.contentWindow.postMessage("<img src=0 onerror=print()>", "*")'></iframe>
	```

### Web Message en URL

- Si el mensaje se vuelca sobre una URL sin una buena sanitización, se puede jugar con el concepto de `javascript:alert()` para ejecutar código javascript como URL.
	Ejemplo con sanitización de requerimiento de 'http:' en la cadena:
	``` html
	<iframe width=600px height=600px src="https://0a1e007204201cb784d9697e006d0056.web-security-academy.net/" 
	onload='this.contentWindow.postMessage("javascript:print()//http:", "*")'>
	</frame>
	```

### Web Message en JSON.parse

- Si el mensaje se pasa por JSON.parse, y los parámetros del json no son validados correctamente, se puede aprovechar para construir un json malicioso para explotar la vulnerabilidad de la *lógica de la web* de la misma forma que con los ejemplos anteriores.

-----------
# DOM-Based Open Redirection

Se puede aprovechar la mala configuración de la página al hacer redirecciones para redireccionar al usuario a otros dominios, como `http://attacker.com`.

Ejemplo de mal redireccionamiento:
``` html
<a href='#' onclick='returnUrl = /url=(https?:\/\/.+)/.exec(location); location.href = returnUrl ? returnUrl[1] : "/"'>Back to Blog</a>
```

-------------
# DOM-Based Cookie Manipulation

Si la página actualiza una cookie via DOM:
``` html
<script>
    document.cookie = 'lastViewedProduct=' + window.location + '; SameSite=None; Secure'
</script>
```

se puede aprovechar para manipular la cookie y forzar acciones no deseadas.

------------
# DOM Clobbering

Es una técnica mediante la cual se sobrescriben *propiedades o métodos preexistentes* en una página web mediante atributos `id` o `name` de elementos html inyectados.

De este modo, si en la lógica de la página se hace referencia a esta propiedad o método sobrescrito, se cargará el elemento creado por el atacante.

*Funciona diferente entre navegadores*

## Para habilitar XSS

- Ejemplo:
	La sección de comentarios de la página permite introducir *html-"safe"*.
	La web carga los avatares de los comentarios de esta manera:
	``` javascript
	let defaultAvatar = window.defaultAvatar || {avatar: '/resources/images/avatarDefault.svg'}
	
	let avatarImgHTML = '<img class="avatar" src="' + (comment.avatar ? escapeHTML(comment.avatar) : defaultAvatar.avatar) + '">';
	```
	Como accede a la variable global *defaultAvatar* para volcar el contenido del atributo *avatar* sobre el *src* de una etiqueta `<img>`, se puede sobrescribir para introducir valores en el src o incluso escapar de ese contexto para inyectar un `onerror="alert()"`.

### Explotación

Se introducen dos elementos con el mismo valor de `id=defaultAvatar`. De esta forma, se está referenciando *defaultAvatar* como un **HTML-Collection**.

Al segundo elemento se le asigna un `name=avatar` con el nombre de la propiedad que se llama. De esta manera, al llamar a `defaultAvatar.avatar` se mostrará la segunda etiqueta.

Para inyectar contenido específico, como javascript, se puede aprovechar la llamada implícita de **toString()** al concatenar cadenas para, mediante una propiedad *href* de un anchor, darle ese valor.

*\*Nota*: En el ejemplo, el comentario se pasa por **DOMPurify.sanitize()**, que hace *url-encode* a las comillas (entre otras cosas), pero permite el protocolo `cid:`, al que no realiza ese url-encode a las comillas.

Con todo esto, se elabora el siguiente payload:
``` html
<a id=defaultAvatar>
<a id=defaultAvatar name=avatar href="cid:0&quot; onerror=alert()>">
```

De esta forma se sobrescribirá la variable `defaultAvatar` para valer la colección html, y cuando se cree el siguiente comentario, la imagen de avatar será:
``` html
<img class="avatar" src="cid:0" onerror="alert()">
```

## DOM attributes

- Se puede realizar un DOM clobbering de atributos para bypassear filtros html.

	-*Ejemplo*:
	
	La página web permite introducir elementos concretos con atributos concretos en la sección de comentarios.
	La comprobación y sanitización del contenido de los comentarios se realiza mediante la librería **HTMLJanitor**.
	Esta librería es vulnerable en este punto del código:
	``` javascript
	// Sanitize attributes
      for (var a = 0; a < node.attributes.length; a += 1) {
        var attr = node.attributes[a];

        if (shouldRejectAttr(attr, allowedAttrs, node)) {
          node.removeAttribute(attr.name);
          // Shift the array to continue looping.
          a = a - 1;
        }
      }
	```
	Se van comprobando y eliminando los atributos en un bucle donde se itera las veces que vale la longitud de *attributes*. Si se realiza un clobbering a esa variable, se puede romper el bucle, haciendo que no se compruebe ningún atributo y saltando así toda la sanitización.

	*\*Nota*: Los elementos permitidos son: `<form>` y `<input>`.

	Para ello, se introduce en el comentario el siguiente contenido:
	``` html
	<form tabindex=0 autofocus=autofocus onload=alert()>
	<input id=attributes>
	```
	De esta forma, *attributes.length* pasa a valer *undefined*, por lo que el bucle se rompe y no se realiza ninguna iteración.