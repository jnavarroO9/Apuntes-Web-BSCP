
---------
# Definición

>Un **JSON Web Token (JWT)** es un **mecanismo de autenticación y transmisión de información firmado digitalmente** que se usa para **identidad y autorización entre sistemas**, normalmente en aplicaciones web y APIs.

>En **pentesting**, un **JWT** se considera **un token autocontenido que representa la identidad o privilegios de un usuario**, y que **puede convertirse en un vector de ataque si su validación, firma o manejo es incorrecto**.

Desde la perspectiva ofensiva, el análisis de JWT se centra en:

1. **Manipulación del token**
    
    - Intentar modificar los **claims** (ej. `role: admin`) para elevar privilegios.
        
    - Verificar si el servidor valida correctamente la firma.
        
2. **Ataques contra el algoritmo**
    
    - Ataques como **`alg: none`** si el servidor acepta tokens sin firma.
        
    - Confusión de algoritmos (ej. **RS256 → HS256**).
        
3. **Exposición de secretos**
    
    - Intentar descubrir el **secret key** si el token usa **HMAC SHA-256 (HS256)** mediante fuerza bruta.
        
4. **Validación incorrecta**
    
    - Revisar si el servidor valida correctamente:
        
        - `exp` (expiración)
            
        - `iss` (issuer)
            
        - `aud` (audience)
            
5. **Problemas de almacenamiento**
    
    - Tokens en **localStorage** vulnerables a **Cross-Site Scripting (XSS)**.

## JWT format

El JWT consiste en 3 partes: cabecera, payload y firma, separadas cada una por un punto:

``` jwt
eyJraWQiOiI5MTM2ZGRiMy1jYjBhLTRhMTktYTA3ZS1lYWRmNWE0NGM4YjUiLCJhbGciOiJSUzI1NiJ9.eyJpc3MiOiJwb3J0c3dpZ2dlciIsImV4cCI6MTY0ODAzNzE2NCwibmFtZSI6IkNhcmxvcyBNb250b3lhIiwic3ViIjoiY2FybG9zIiwicm9sZSI6ImJsb2dfYXV0aG9yIiwiZW1haWwiOiJjYXJsb3NAY2FybG9zLW1vbnRveWEubmV0IiwiaWF0IjoxNTE2MjM5MDIyfQ.SYZBPIBg2CRjXAJ8vCER0LA_ENjII1JakvNQoP-Hw6GG1zfl4JyngsZReIfqRvIAEi5L4HV0q7_9qGhQZvy9ZdxEJbwTxRs_6Lb-fZTDpW6lKYNdMyjw45_alSCZ1fypsMWz_2mTpQzil0lOtps5Ei_z7mM7M8gCwe_AGpI53JxduQOaB5HkT5gVrv9cKu9CsW5MS6ZbqYXpGyOG5ehoxqm8DL5tFYaW3lB50ELxi0KsuTKEbD0t5BCl0aCR2MBJWAbN-xeLwEenaqBiwPVvKixYleeDQiBEIylFdNNIMviKRgXiYuAvMziVPbwSgkZVHeEdF5MQP1Oe2Spac-6IfA
```

La cabecera y el payload son solo objetos JSON base64url-encodeados. La cabecera contiene metadata sobre el propio token, mientras que el payload contiene los claims del usuario:

``` json
{
    "iss": "portswigger",
    "exp": 1648037164,
    "name": "Carlos Montoya",
    "sub": "carlos",
    "role": "blog_author",
    "email": "carlos@carlos-montoya.net",
    "iat": 1516239022
}
```

## JWT signature

El servidor que construye el token típicamente genera una firma hasheando la cabecera y el payload; en algunos casos encriptando también el hash resultante. Este proceso requiere una clave secreta de firma. Este mecanismo permite a los servidores verificar si los datos del token han sido modificados desde que fue emitido:

- Como la firma se deriva directamente del resto del token, cualquier cambio (un simple byte) de la cabecera o el payload resultará en una falta de coincidencia con la firma.

- Sin conocer la clave secreta de firma, no debería ser posible generar la firma correcta para una cabecera o payload.

## JWT vs JWS vs JWE

![[JWT-JWS-JWE_JWT.png]]

Los JSON Web Signatures (JWS) y los JSON Web Encrytions (JWE) heredan de la especificación JWT.

Los JWS son los que habitualmente se refiere cuando se habla de JWT. Los JWE son muy similares, salvo que el contenido del token está encriptado en vez de codificado.

----------
# Exploiting flawed JWT signature verification

Por diseño, los servidores no almacenan ninguna información sobre los JWTs que emite. En cambio, cada token almacena toda la información. Esto introduce un problema, y es que si el servidor no verifica correctamente la firma, cualquier atacante puede modificar el contenido del resto del token sin ninguna restricción.

Por ejemplo, considerando los siguientes claims de un JWT:

``` json
{
    "username": "carlos",
    "isAdmin": false
}
```

Si el servidor identifica la sesión basándose en este `username`, modificando este parámetro permite la suplantación de otros usuarios. Por otro lado, si el valor de `isAdmin` se utiliza para control de acceso, puede ser un vector para escalar privilegios.

## Accepting arbitrary signatures

Las librerías JWT típicamente proporcionan un método para verificar los tokens y otro solo para decodificarlos. Por ejemplo, la librería `jsonwebtoken` de Node.js tiene `verify()` y `decode()`.

Ocasionalmente, los desarrolladores confunden estos dos métodos, y solo pasanel token por el método `decode()`. Esto significa que la aplicación no verifica la firma en ningún momento.

## Accepting tokens with no signature

En la cabecera del JWT existe un parámetro `alg` que especifica que algoritmo se ha utilizado para firmar el token.

``` json
{
    "alg": "HS256",
    "typ": "JWT"
}
```

Esto es de base vulnerable ya que el servidor no tiene otra opción más que confiar en input del token controlado por el usuario que, en este punto, todavía no ha sido verificado.

Los JWT se pueden firmar con diferentes algoritmos, pero también se pueden dejar sin firmar. En ese caso, el parámetro `alg` vale `none`. Los servidores usualmente rechazan tokens sin firma. Sin embargo, como este filtro se basa en *string parsing*, se pueden probar técnicas de ofuscación para bypassear  el filtro:

- Mezclar mayúsculas y minúsculas
- Utilizar codificaciones inesperadas.
- ...

*\*Nota*: Aunque el token no tenga firma, el payload debe acabar con un punto final.

--------------
# Brute-forcing secret keys

Algunos algoritmos utilizan una clave secreta para generar la firma. Es importante que esta sea difícil de adivinar por fuerza bruta, ya que si un atacante la consigue adivinar, puede forjar sus propios tokens con firmas válidas.

Para hacer fuerza bruta, es buena opción utilizar el siguiente diccionario que contiene secretos de códigos snippet de internet, secretos placeholder, y por defecto.

- [wordlist of well-known secrets](https://github.com/wallarm/jwt-secrets/blob/master/jwt.secrets.list)

## Brute-forcing secret keys using hashcat / john

Se descarga el JWT y se introduce en un fichero para hacer fuerza bruta en local.

### Hashcat

``` bash
hashcat -a 0 -m 16500 <jwt> <wordlist>
```

### John The Ripper

``` bash
john <jwt> --wordlist=<wordlist>
```

*\*Nota*: Ambos guardan los hashes que consiguen crackear junto con el hash crackeado resultante, por lo que si se intenta romper un hash ya obtenido, no se mostrará el resultado. Para verlo, es necesario incluir la flag `--show`.

Si el servidor utiliza un secreto extremadamente débil, puede ser posible hacer fuerza bruta caracter por caracter en vez de usar un diccionario.

----------
# JWT header parameter injections

En la especificación JWS, solo el parámetro de la cabecera `alg` es obligatorio. Pero en la práctica, las cabeceras JWT suelen contener otros parámetros. Los siguientes son de particular interés para los atacantes:

- `jwk` (JSON Web Key) - Proporciona un objeto JSON embebido representando la clave.
- `jku` (JSON Web Key Set URL) - Proporciona una URL donde los servidores pueden buscar un conjunto de claves que contiene la clave correcta.
- `kid` (Key ID) - Proporciona un ID que los servidores pueden utilizar para identificar la clave correcta en casos donde haya múltiples claves para escoger. Dependiendo del formato de la clave, puede tener un parámetro `kid` coincidente.

Estos parámetros indican al servidor qué clave pública debe utilizar para verificar la firma del JWT.

## Injecting self-signed JWTs via the jwk parameter

Ejemplo de un JWK en una cabecera JWT:

``` json
{
    "kid": "ed2Nf8sb-sD6ng0-scs5390g-fFD8sfxG",
    "typ": "JWT",
    "alg": "RS256",
    "jwk": {
        "kty": "RSA",
        "e": "AQAB",
        "kid": "ed2Nf8sb-sD6ng0-scs5390g-fFD8sfxG",
        "n": "yy1wpYmffgXBxhAUJzHHocCuJolwDqql75ZWuCQ_cb33K2vh9m"
    }
}
```

Idealmente, los servidores deberían usar solo una lista blanca limitada de claves públicas para verificar firmas JWT. Sin embargo, servidores mal configurados a veces utilizan cualquier clave que esté embebida en el parámetro `jwk`.

Este comportamiento se puede explotar firmando el JWT modificado con una clave privada RSA propia, y embebiendo el par de clave pública en la cabecera `jwk`.

Es importante que el parámetro `kid` de la cabecera coincida con el `kid` del JWK.

## Injecting self-signed JWTs via the jku parameter

Un JWK Set es un objeto JSON que contiene un array de JWKs representando diferentes claves.

``` json
{
    "keys": [
        {
            "kty": "RSA",
            "e": "AQAB",
            "kid": "75d0ef47-af89-47a9-9061-7c02a610d5ab",
            "n": "o-yy1wpYmffgXBxhAUJzHHocCuJolwDqql75ZWuCQ_cb33K2vh9mk6GPM9gNN4Y_qTVX67WhsN3JvaFYw-fhvsWQ"
        },
        {
            "kty": "RSA",
            "e": "AQAB",
            "kid": "d8fDFo-fS9-faS14a9-ASf99sa-7c1Ad5abA",
            "n": "fc3f-yy1wpYmffgXBxhAUJzHql79gNNQ_cb33HocCuJolwDqmk6GPM4Y_qTVX67WhsN3JvaFYw-dfg6DH-asAScw"
        }
    ]
}
```

Los JWK Sets están algunas veces expuestos públicamente a través de un endpoint estándar, como `/.well-known/jwks.json`.

Las páginas web más seguras solo buscan claves de dominios confiables, pero se pueden utilizar técnicas de discrepancia en el parsing URL para bypassear este filtro.

## Injecting self-signed JWTs via the kid parameter

Las claves de verificación habitualmente se almacenan en un JWK Set y utiliza el `kid` para identificar la correcta. Sin embargo, la especificación JWS no define una estructura concreta del ID, es solo un string arbitrario que han elegido los desarrolladores.

Por ejemplo, puede que utilicen el parámetro `kid` para apuntar a una entrada particular de la base de datos, o incluso a un archivo del sistema.

Si este parámetro es también vulnerable a *directory traversal*, se puede forzar al servidor a usar un archivo arbitrario de su sistema de archivos para verificar la clave.

``` json
{
    "kid": "../../path/to/file",
    "typ": "JWT",
    "alg": "HS256",
    "k": "asGsADas3421-dfh9DGN-AFDFDbasfd8-anfjkvc"
}
```

Esto es especialmente peligroso si el servidor soporta JWTs firmados usando algoritmos simétricos. En este caso, se puede introducir un `kid` de un archivo predecible y firmar el JWT con el contenido de ese archivo.

La forma más fácil de hacerlo es apuntando al archivo `/dev/null`, que devuelve una cadena vacía. De este modo, tan solo hay que firmar el JWT con un string vacío.

Si el servidor guarda sus claves de verificación en la base de datos, el parámetro `kid` puede ser un potencial vector para *ataques de inyección SQL*.

## Other interesting JWT header parameters

- `cty` (Content Type) - Usado para declarar el tipo de medio para el contenido del payload del JWT. Se omite normalmente de la cabecera, pero la librería que parsea el token puede que lo soporte.

	Si se ha encontrado una forma de bypassear la verificación de la firma, se puede provar a inyectar el parámetro `cty` para cambiar el content type a `text/xml` o `application/x-java-serialized-object`, lo que potencialmente puede habilitar vectores de *ataques XXE y deserialización*.

- `x5c` (X.509 Certificate Chain) - Usado para pasar el certificado de clave pública X.509 o el certificate chain de la clave usada para firmar digitalmente el JWT. Este parámetro se puede utilizar para inyectar certificados firmados propios, similar a los ataques de inyección de parámetro `jwk`.

	Debido a la complejidad del formato X.509 y sus extensiones, parsear estos certificados puede introducir vulnerabilidades. 

	- CVE-2017-2800
	- CVE-2018-2633

--------
# JWT algorithm confusion

>Ocurre cuando un atacante puede forzar al servidor a verificar la firma del JWT usando un algoritmo diferente al intencionado por los desarrolladores. Puede permitir a los atacantes forjar JWTs conteniendo valores arbitrarios sin necesidad de conocer la clave de firma secreta.

## Symmetric vs asymmetric algorithms

- Los algoritmos **simétricos** como *HS256* (HMAC + SHA-256) utilizan la misma clave para firmar como para verificar el token.

![[symmetric-key_JWT.png]]

- Los algoritmos **asimétricos** como *RS256* (RSA + SHA-256), en cambio, utilizan un par de claves asimétrico, que consiste en una clave privada para firmar el token, y otra clave pública relacionada matemáticamente con la anterior que sirve para verificar la firma.

![[asymmetric-key_JWT.png]]

Como el nombre sugiere, la clave privada debe permanecer secreta, pero la clave pública usualmente se comparte para que cualquiera pueda verificar la firma de los tokens emitidos por el servidor.

## How do algorithm confusion vulnerabilities arise?

Las vulnerabilidades de confusión de algoritmo típicamente se originan por fallas en implementación de las librerías JWT. Aunque el proceso de verificación de la firma difiere según el algoritmo utilizado, muchas librerías proporcionan un único método para verificarlas. Estos métodos se basan en el parámetro `alg` de la cabecera del token para determinar que tipo de verificación deben realizar.

El siguiente pseudo-código muestra un ejemplo simplificado de la declaración de este método genérico `verify()` que se podría encontrar en una librería JWT:

``` js
function verify(token, secretOrPublicKey){
    algorithm = token.getAlgHeader();
    if(algorithm == "RS256"){
        // Use the provided key as an RSA public key
    } else if (algorithm == "HS256"){
        // Use the provided key as an HMAC secret key
    }
}
```

Los problemas surgen cuando los desarrolladores asumen que este método solo tratará JWTs firmados usando un algoritmo asimétrico, por lo que pasan siempre una clave pública fija al método de esta manera:

``` js
publicKey = <public-key-of-server>;
token = request.getCookie("session");
verify(token, publicKey);
```

En este caso, si el servidor recibe un token firmado usando un algoritmo simétrico como HS256, el método `verify()` tratará la clave pública como el secreto HMAC.

Esto quiere decir que se puede firmar un token usando HS256 y la clave pública, y el servidor utilizará la misma clave pública para verificar la firma.

## Performing an algorithm confusion attack

### 1. Obtener la clave pública del servidor

Hay dos opciones:

#### Claves públicas expuestas

Las claves públicas que utiliza el servidor para verificar firmas están expuestas en un endpoint estándar:

- `/jwks.json`
- `/.well-known/jwks.json`

#### Clave pública derivada de tokens existentes

Se puede tratar de derivar la clave pública utilizada para verificar los tokens en base a dos JWT, mediante la herramienta `jwt_forgery.py` del repositorio [rsa_sign2n](https://github.com/silentsignal/rsa_sign2n).

``` bash
python3 jwt_forgery.py <jwt1> <jwt2>
```

El comando devuelve:

- una clave PEM en X.509 y en PKCS1 codificadas a base64,
- un JWT firmado usando cada una de las claves

para cada potencial valor de `n` calculado. Hay que probar los tokens con distinta `n` para averiguar cuál coincide con el valor de `n` en el servidor. Solo ese será válido en la página.

### 2. Convertir la clave pública al formato adecuado

Es necesario convertir la clave obtenida al formato exacto que tiene la clave pública almacenada en el servidor. Aunque el servidor exponga las claves en formato JWK, puede que guarde las claves en formato PEM X.509.

Para llevarlo a cabo, es útil la extensión **JWT Editor Keys** de Burpsuite, siguiendo los siguientes pasos:

1. With the extension loaded, in Burp's main tab bar, go to the *JWT Editor Keys* tab.

2. Click *New RSA* Key. In the dialog, paste the JWK that you obtained earlier.

3. Select the *PEM* radio button and copy the resulting PEM key.

4. Go to the *Decoder* tab and Base64-encode the PEM.

5. Go back to the JWT Editor Keys tab and click *New Symmetric Key*.

6. In the dialog, click *Generate* to generate a new key in JWK format.

7. Replace the generated value for the ``k`` parameter with a Base64-encoded PEM key that you just copied.

8. Save the key.


### 3. Crear el JWT malicioso con un payload modificado y el parámetro `alg` con valor `HS256`

Una vez obtenida la clave pública en un formato adecuado, se modifica el JWT. A parte de modificar el payload, es necesario cambiar el parámetro `alg` a `HS256`.

### 4. Firmar el token con HS256, usando la clave pública como secreto

Se firma el token usando el algoritmo HS256 con la clave pública RSA como secreto.