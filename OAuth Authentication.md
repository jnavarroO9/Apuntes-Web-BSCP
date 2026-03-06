
----------

# Definición

>**OAuth (Open Authorization)** es un **protocolo de autorización delegada** que permite a una aplicación obtener acceso limitado a recursos de un usuario en otro servicio **sin compartir sus credenciales (usuario y contraseña)**.

Desde el punto de vista técnico, OAuth define un flujo basado en **tokens de acceso** emitidos por un **servidor de autorización**, que el cliente utiliza para consumir APIs protegidas.

En el contexto de **red teaming, pentesting o bug bounty**, OAuth no se analiza como un mecanismo “seguro por defecto”, sino como una **superficie de ataque crítica**

Desde la ofensiva, OAuth es interesante porque:

1. **No protege autenticación directamente**, sino autorización.
    
2. Se basa en **redirecciones HTTP**, lo que abre la puerta a:
    
    - Open Redirect
        
    - Token leakage
        
    - Authorization code interception
        
3. Depende fuertemente de:
    
    - Validación estricta de `redirect_uri`
        
    - Uso correcto de `state`
        
    - Implementación segura de PKCE

------------
# OAuth grant types / OAuth flows

>Determinan la secuencia exacta de pasos del proceso de autenticación OAuth.

Los dos tipos más comunes son:

## Authorization code

![[AuthorizationCode_OAuth.png]]

### 1. Authorization request

La aplicación cliente manda una petición al endpoint `/authorization` del servicio OAuth solicitando permiso para acceder a datos específicos del usuario. Lo realiza a través del usuario, redirigiéndolo.

``` http
GET /authorization?client_id=12345&redirect_uri=https://client-app.com/callback&response_type=code&scope=openid%20profile&state=ae13d489bd00e3c24 HTTP/1.1
Host: oauth-authorization-server.com
```

Esta petición contiene los siguientes parámetros, normalmente proporcionados en la query string:

- `client_id`
	Identificador de la aplicación cliente. El valor se genera cuando la aplicación cliente se registra en el servicio OAuth.

- `redirect_uri`
	La URI a la que el navegador del usuario debe ser redirigido cuando se mande el código de autorización a la aplicación cliente. También se conoce como "callback URI" o "callback endpoint".

- `response_type`
	Determina el tipo de respuesta que espera la aplicación cliente, por tanto, que flow quiere iniciar. Para *authorization code grant type*, el valor debe ser `code`.

- `scope`
	Especifica a qué subconjunto de datos del usuario quiere acceder la aplicación cliente.
	*\*Nota*: Los scopes pueden ser custom, indicados por el proveedor OAuth, o estandarizados definidos por la especificación **OpenID Connect**.

- `state`
	Guarda un valor único e impredecible que está vinculado a la sesión actual en la aplicación cliente. El servicio OAuth debe devolver exactamente el mismo valor en la respuesta, junto con el código de autorización. Este parámetro sirve como si fuera un token CSRF para la aplicación cliente, asegurándose que la petición a su endpoint `/callback` sea de la misma persona que inició el OAuth flow.

### 2. User login and consent

Cuando el servidor OAuth recibe la petición inicial, redirige al usuario a la página de login, donde se le pide iniciar sesión a su cuenta con el proveedor OAuth. Por ejemplo con su cuenta de una red social.

Después se le presenta una lista de datos que la aplicación cliente quiere acceder. Estos se basan en los scopes indicados en la [[#1. Authorization request]]. El usuario puede elegir si consentir o no el acceso.

Una vez aceptados los scopes para la aplicación cliente, este paso se realizará de forma automática las próximas veces, siempre y cuando el usuario aún tenga una sesión válida en el servicio OAuth.

### 3. Authorization code grant

Si el usuario consiente los accesos solicitados, su navegador será redirigido al endpoint `/callback` especificado en el parámetro `redirect_uri`.

La petición `GET` resultante contendrá el código de autorización como un parámetro de la query. Dependiendo de la configuración, también puede incluirse el parámetro `state` con el mismo valor que en la authorization request.

``` http
GET /callback?code=a1b2c3d4e5f6g7h8&state=ae13d489bd00e3c24 HTTP/1.1
Host: client-app.com
```

### 4. Access token request

Una vez que la aplicación cliente recibe el código de autorización, necesita intercambiarlo por un token de acceso. Para hacerlo, manda una petición `POST` server-to-server al endpoint `/token` del servicio OAuth.

Toda la comunicación de aquí en adelante se realiza por un canal seguro, por lo que usualmente no se puede observar ni controlar por un atacante.

``` http
POST /token HTTP/1.1
Host: oauth-authorization-server.com
…
client_id=12345&client_secret=SECRET&redirect_uri=https://client-app.com/callback&grant_type=authorization_code&code=a1b2c3d4e5f6g7h8
```

En esta petición aparecen nuevos parámetros:

- `client_secret`
	La aplicación cliente debe autorizarse incluyendo la clave secreta que se le asignó cuando se registró en el servicio OAuth.

- `grant_type`
	Usado para que el nuevo endpoint conozca qué grant type quiere utilizar la aplicación cliente. En este caso, el valor debe ser `authorization_code`.

### 5. Access token grant

El servicio OAuth valida el token de acceso de la petición. Si todo es correcto, el servidor responde proporcionando a la aplicación cliente un token de acceso con el scope solicitado.

``` json
{
    "access_token": "z0y9x8w7v6u5",
    "token_type": "Bearer",
    "expires_in": 3600,
    "scope": "openid profile",
    …
}
```

### 6. API call

Ahora que la aplicación cliente tiene el código de acceso, puede finalmente buscar los datos del usuario del servidor de recursos. Para hacer esto, hace una llamada a la API al endpoint `/userinfo` del servicio OAuth. el token de acceso se manda a través de la cabecera `Authorization: Bearer`, para demostrar que la aplicación cliente tiene permiso para acceder a esos datos.

``` http
GET /userinfo HTTP/1.1
Host: oauth-resource-server.com
Authorization: Bearer z0y9x8w7v6u5
```

### 7. Resource grant

El servidor de recursos debe verificar que el token sea válido y que pertenezca a la aplicación cliente actual. Si es así, responde mandando los recursos solicitados, por ejemplo, los datos del usuario basados en el scope del token de acceso.

``` json
{
    "username":"carlos",
    "email":"carlos@carlos-montoya.net",
    …
}
```

La aplicación cliente finalmente puede utilizar estos datos para su propósito intencionado. En el caso de OAuth authentication, típicamente será usado como ID para proporcionar al usuario una sesión autenticada, efectivamente iniciándoles sesión.

## Implicit grant type

Es mucho más simple. En vez de conseguir un código de autorización para después cambiarlo por un token de acceso, la aplicación cliente recibe el token de acceso directamente después de que el usuario dé el consentimiento.

Este flow es mucho menos seguro, ya que toda la comunicación se realiza a través de redirecciones del navegador, por lo que el token de acceso y los datos del usuario están más expuestos a potenciales ataques.

![[Implicit_OAuth.png]]

### 1. Authorization request

El implicit flow empieza de la misma manera que el authorization flow. La única diferencia está en el parámetro `response_type`, que debe tener el valor `token`.

``` http
GET /authorization?client_id=12345&redirect_uri=https://client-app.com/callback&response_type=token&scope=openid%20profile&state=ae13d489bd00e3c24 HTTP/1.1
Host: oauth-authorization-server.com
```

### 2. User login and consent

El usuario inicia sesión y decide si consentir o no los permisos solicitados. Exactamente igual que el authorization flow.

### 3. Access token grant

Si el usuario da consentimiento, es donde las cosas empiezan a diferir. El servicio OAuth redirige al navegador del usuario al `redirect_uri` especificado en la authorization request. Sin embargo, en vez de mandar el token en un parámetro de la query, lo manda como un fragmento URL.

``` http
GET /callback#access_token=z0y9x8w7v6u5&token_type=Bearer&expires_in=5000&scope=openid%20profile&state=ae13d489bd00e3c24 HTTP/1.1
Host: client-app.com
```

La aplicación cliente debe utilizar un script para extraer el fragmento y almacenarlo.

### 4. API call

Una vez que la aplicación cliente extrae el token de acceso del fragmento URL, puede usarlo para hacer llamadas al endpoint `/userinfo` del servicio OAuth. A diferencia del authorization code flow, esto también pasa a través del navegador.

### 5. Resource grant

El servidor de recursos verifica que el token es válido y que pertenece a la sesión actual de la aplicación cliente. Entonces, responde con los datos del usuario solicitados en el scope.

``` json
{
    "username":"carlos",
    "email":"carlos@carlos-montoya.net"
}
```

La aplicación cliente ya puede utilizar los datos para el propósito intencionado.

----------
# Identify and recon

Para identificar que la aplicación está usando OAuth authentication, tan solo hay que prestar atención a la manera en la que se inicia sesión en la aplicación. Si se realiza una petición a un endpoint `/authorization` con los parámetros `client_id`, `redirect_uri` y `response_type`, se trata de un inicio de flow OAuth.

Para hacer reconocimiento, primero hay que identificar el dominio del servicio OAuth, además de analizar las peticiones que se realizan en el flow.

Como estos servicios proporcionan una API pública, a veces hay documentación disponible que puede resultar de ayuda, como nombres exactos de endpoints, y que opciones de configuración se están utilizando.

Teniendo el hostname del servidor de autorización, siempre hay que probar a mandar peticiones `GET` a estos endpoints:

- `/.well-known/oauth-authorization-server`
- `/.well-known/openid-configuration`

Esto a veces devuelve un archivo de configuración JSON con información clave, como detalles de funcionalidades adicionales que estén soportadas.

---------
# OAuth authentication vulnerabilities

>Las aplicaciones cliente normalmente utilizarán servicios OAuth reputados, que tendrán protecciones ante exploits conocidos. Sin embargo, su parte de la implementación puede ser menos segura.

## Improper implementation of the implicit grant type

En este flow, la aplicación cliente recibe el token de acceso a través del fragmento URL en el endpoint `/callback`. Sin embargo, si la aplicación quiere mantener la sesión después de que el usuario cierre la página, necesita guardar los datos del usuario (normalmente el ID de usuario y el token de acceso) en algún lugar.

Para ello, la aplicación cliente normalmente mandará esta información al servidor en una petición `POST`, que asigna la cookie de sesión. Esta petición es prácticamente equivalente a un formulario clásico de usuario-contraseña, pero en este caso se confía en la información que manda el usuario-

En el implicit flow, esta petición está expuesta para los atacantes a través de sus navegadores. Esto puede ser vulnerable si la aplicación cliente no valida correctamente que el token de acceso coincide con los otros datos de la petición. En este caso, un atacante puede simplemente cambiar los parámetros para hacerse pasar por otro usuario.

``` http
POST /me HTTP/2
Host: vulnerable.com
...
{
	"username":"carlos",
    "email":"carlos@carlos-montoya.net",
    "token":"token-de-josmo"
}
```

``` http
HTTP/2 200 OK
Set-Cookie: session=cookie-de-carlos
```

## Flawed CSRF protection

Es el caso en el que la petición de autorización no contiene el parámetro `state`, o su valor es predecible. Esto quiere decir que un atacante puede iniciar el OAuth flow, y hacer que el navegador de otro usuario lo complete. Puede tener consecuencias severas dependiendo de cómo se esté utilizando OAuth en la aplicación cliente.

Considerando una página que permite a los usuarios iniciar sesión usando tanto usuario-contraseña, como vinculando un perfil de red social a su cuenta mediante OAuth. Si la aplicación no utiliza el parámetro `state`, un atacante puede secuestrar la cuenta de la víctima vinculando el perfil de red social del atacante a la cuenta de la víctima.

## Leaking authorization codes and access tokens

Se basa en hacer que el usuario víctima inicie el OAuth flow, pero modificar el valor del parámetro `redirect_uri` para que el código de autorización o el token de acceso los reciba el atacante en vez del endpoint `/callback` de la aplicación cliente.

Con el código de autorización o token de acceso, el atacante puede iniciar sesión como el usuario víctima.

*\*Nota*: Usar protecciones `state` o `nonce` no necesariamente previene estos ataques, ya que el atacante puede generar nuevos valores con su propio navegador.

Servidores de autorización más seguros pedirán un parámetro `redirect_uri` cuando se intercambia el código de acceso con el token de acceso en el endpoint `/token`. Verificarán que el valor sea el mismo que el proporcionado en la petición de autorización, y si no coinciden, se rechaza. Como esta comunicación es server-to-server, el atacante no puede controlar el segundo parámetro `redirect_uri`.

### Flawed redirect_uri validation

Es una buena práctica para las aplicaciones clientes proporcionar una lista blanca de callback URIs cuando se registra en el servicio OAuth. De esta manera, cualquier valor de `redirect_uri` ajeno a esta lista resultará en un error. Sin embargo, aún pueden haber formas de bypassear la validación.

Es importante experimentar con el parámetro `redirect_uri` para entender cómo se está validando:

- Algunas implementaciones permiten un rango de subdirectorios comprobando solo que el string empiece por la secuencia correcta de caracteres, por ejemplo, el dominio registrado. Se debe probar a cambiar el **path**, **parámetros de la query**, y **fragmentos**.

- Si se pueden añadir valores extra al final del valor por defecto del parámetro `redirect_uri`, pueden ser explotables **discrepancias entre el parsing** de la URI de diferentes componentes del servicio OAuth. Se pueden probar técnicas como:
	`https://default-host.com &@foo.evil-user.net#@bar.evil-user.net/`

- Puede que haya vulnerabilidades **server-side parameter pollution**. En este caso, se puede tratar de enviar el parámetro `redirect_uri` duplicado.
	``` URL
	https://oauth-authorization-server.com/?client_id=123&redirect_uri=client-app.com/callback&redirect_uri=evil-user.net
	```

- Algunos servidores dan un trato especial a las URIs `localhost`, ya que suelen utilizarse en desarrollo. En algunos casos, cualquier URI que comience con `localhost` puede estar permitida en el entorno de producción. Esto puede permitir bypassear la validación, al registrar un dominio como `localhost.evil-user.net`.

Es importante también modificar otros parámetros, ya que pueden alterar la forma en la que el servicio OAuth valida el parámetro `redirect_uri`. Por ejemplo, cambiar el valor de `response_mode` de `query` a `fragment` puede alterar el parsing de `redirect_uri`.

También, si el modo de respuesta `web_message` está disponible, permite un rango más grande de subdominios en `redirect_uri`.

### Stealing codes and access tokens via proxy page

Cuando no se consigue de ninguna manera introducir un dominio externo en el parámetro `redirect_uri`, se puede tratar de introducir otras páginas del dominio de la lista blanca.

Normalmente, no habrá ningún subdirectorio interesante en el path del callback `/oauth/callback`. Sin embargo, se puede probar con un path traversal para introducir un path arbitrario:

`https://client-app.com/oauth/callback/../../example/path`

El backend puede interpretarlo como:

`https://client-app.com/example/path`

Una vez identificadas otras páginas que se pueden introducir, se deben auditar en busca de vulnerabilidades adicionales que permitan potencialmente filtrar el código de autorización o el token de acceso.

La vulnerabilidad más útil para ello es el **open redirect**. Con esto, puedes utilizar la página como un proxy para redirigir los tokens al dominio controlado por el atacante.

*\*Nota*: Con el implicit flow, robar el token de acceso no solo permite iniciar sesión en la cuenta de la víctima, sino también hacer llamadas a la API del servidor de recursos del servicio OAuth. Esto puede revelar información sensible del usuario que normalmente no sería accesible desde la interfaz de usuario de la aplicación cliente.

También se pueden buscar otras vulnerabilidades que permitan la extracción del token a un dominio externo:

- JavaScript que trata query parameters y URL fragments
- XSS vulnerabilities
- HTML injection vulnerabilities

## Flawed scope validation

En ocasiones es posible "mejorar" un token de acceso (bien robado u obtenido usando una aplicación cliente maliciosa) con permisos extra debido a la poca validación del servicio OAuth. El proceso para llevarlo a cabo depende del tipo de flow.

### Scope upgrade: authorization code flow

En este tipo de flow, los datos del usuario se solicitan a través de la comunicación segura server-to-server, que no puede ser manipulada directamente por un atacante. Sin embargo, puede llegar a obtenerse el mismo resultado *registrando una aplicación cliente propia* en el servicio OAuth.

Por ejemplo, la aplicación cliente maliciosa del atacante solicita acceso al email del usuario usando el scope `openid email`. Después de que el usuario acepte la petición, la aplicación cliente recibe el token de autorización. Como el atacante controla la aplicación cliente, puede añadir otro parámetro `scope` a la petición de intercambio código-token con el valor adicional `profile`:

``` http
POST /token
Host: oauth-authorization-server.com
…
client_id=12345&client_secret=SECRET&redirect_uri=https://client-app.com/callback&grant_type=authorization_code&code=a1b2c3d4e5f6g7h8&scope=openid%20 email%20profile
```

Si el servidor no valida si el `scope` es el mismo que el de la petición de autorización, puede que genere un token de acceso usando el nuevo `scope` y lo envíe a la aplicación cliente del atacante:

``` json
{
    "access_token": "z0y9x8w7v6u5",
    "token_type": "Bearer",
    "expires_in": 3600,
    "scope": "openid email profile",
    …
}
```

El atacante puede entonces usar su aplicación para hacer las llamadas a la API necesarias para acceder a los datos del perfil del usuario.

### Scope upgrade: implicit flow

Como el token de acceso se envía a través del navegador, el atacante puede robarlos y utilizarlos directamente para hacer peticiones al endpoint `/userinfo` del servicio OAuth, añadiendo manualmente el parámetro `scope` en el proceso.

El atacante obtendrá la información si el servicio OAuth no valida el valor de `scope` en referencia al de la petición de autorización, y el `scope` introducido no excede los permisos proporcionados a la aplicación cliente.

## Unverified user registration

Las aplicaciones cliente asumen que los datos almacenados por el proveedor OAuth son correctos. Sin embargo, algunas páginas web que ofrecen servicio OAuth permiten a los usuarios registrarse sin verificar todos los detalles, incluyendo el email en algunos casos.

Un atacante puede explotar esto registrando una cuenta en el proveedor OAuth usando los mismos detalles que el usuario víctima. La aplicación cliente puede permitir iniciar sesión en la cuenta de la víctima con la cuenta fraudulenta del proveedor OAuth.

---------
# OpenID Connect

>OpenID Connect solo añade `scopes` estandarizados que son iguales para todos los proveedores, además de un tipo de respuesta extra: `id_token`.

## OpenID Connect scopes

El parámetro `scope` empieza con `openid`, seguido de uno o más `scopes`, también estandarizados:

- `profile`
- `email`
- `address`
- `phone`

Cada uno de ellos permite acceder a un subconjunto de datos del usuario.

## ID token

Devuelve un JSON web token (JWT) firmado con un JSON web signature (JWS). El JWT payload contiene una lista de claims basada en el `scope` inicialmente solicitado.

Este tipo de respuesta reduce las peticiones mandadas, ya que se basa en la firma criptográfica JWT. Sin embargo, algunos ataques aún son posibles, ya que las claves para la verificación de la firma se transmiten por el mismo canal (normalmente expuesto en `/.well-known/jwks.json`).

Este tipo de respuesta se puede combinar con los otros dos, resultando en el servidor OAuth devolviendo tanto el token pertinente como el JWT:

``` text
response_type=id_token token
response_type=id_token code
```

## Identify OpenID Connect

Se puede saber si el servicio OAuth soporta OpenID comprobando si el `scope` de la petición de autorización contiene el valor `openid`.

Si no lo contiene, también se puede probar a incluirlo.

También es interesante intentar acceder a la documentación para ver si hay información útil sobre el soporte de OpenID Connect, en el endpoint `/.well-known/openid-configuration`.

## OpenID Connect Vulnerabilities

### Unprotected dynamic client registration

La forma de registrar un cliente en el proveedor OpenID es mandando una petición `POST` al endpoint `/registration` (el nombre de este endpoint se encuentra normalmente en la documentación).

``` http
POST /openid/register HTTP/1.1
Content-Type: application/json
Accept: application/json
Host: oauth-authorization-server.com
Authorization: Bearer ab12cd34ef56gh89

{
    "application_type": "web",
    "redirect_uris": [
        "https://client-app.com/callback",
        "https://client-app.com/callback2"
        ],
    "client_name": "My Application",
    "logo_uri": "https://client-app.com/logo.png",
    "token_endpoint_auth_method": "client_secret_basic",
    "jwks_uri": "https://client-app.com/my_public_keys.jwks",
    "userinfo_encrypted_response_alg": "RSA1_5",
    "userinfo_encrypted_response_enc": "A128CBC-HS256",
    …
}
```

El proveedor OpenID debería requerir a la aplicación cliente autenticarse, por ejemplo, a través de un token Bearer, como en el ejemplo. Sin embargo, si no lo requiere, un atacante puede registrar su aplicación cliente maliciosa.

Esto permite ataques como SSRF de segundo orden, si no se han tomado las medidas de seguridad pertinentes, accediendo a los recursos de la aplicación víctima a través del proveedor OpenID.

### Allowing authorization requests by reference

Algunos proveedores OpenID dan la opción de pasar la petición de autorización en un JWT. Si esta funcionalidad está soportada, se puede mandar un único parámetro `request_uri` apuntando a un JWT que contiene el resto de parámetros OAuth y sus valores. Dependiendo de la configuración, este parámetro `request_uri` puede ser otro potencial vector para SSRF.

También puede ser útil para bypasear posibles validaciones que actúen sobre la query string pero fallen en el JWT.

Para ver si esta funcionalidad está soportada, hay que mirar por la opción `request_uri_parameter_supported` en el archivo de configuración y en la documentación. A veces, aunque no se mencione explícitamente, el proveedor puede soportarlo.