
---------
# Definición

>Una **race condition** (condición de carrera) en ciberseguridad es una vulnerabilidad que ocurre cuando el comportamiento de un sistema depende del *orden* o el *tiempo* en que se ejecutan múltiples procesos o hilos, y un atacante logra interferir en ese orden para provocar un resultado no deseado.

Se produce cuando un sistema:

1. **Comprueba** una condición (por ejemplo, si un usuario tiene permisos), y
2. **Actúa** después basándose en esa comprobación,

pero entre esos dos pasos un atacante consigue cambiar el estado del sistema.

El periodo de tiempo donde se puede producir una colisión se conoce como *"race window"*.

--------
# Limit overrun race conditions

>Son un tipo de race condition que permite exceder algún límite impuesto por la lógica de negocio de la aplicación.

Por ejemplo, en el caso de una tienda online que permite usar un código de descuento de un solo uso al proceder al pago. Para aplicar el descuento, realizará los siguientes pasos de alto nivel (poco detallados):

1. Comprobar que el descuento no se ha usado.
2. Aplicar el descuento al pedido total.
3. Actualizar el registro de la base de datos para reflejar que ahora se ha usado el código.

Si se quisiera reutilizar el código, el primer paso lo descartaría.

![[Limit-Overrun-Sequential_RaceCond.png]]

Ahora, considerando que un usuario que no ha utilizado el código intenta aplicarlo dos veces casi al mismo tiempo:

![[Limit-Overrun-Concurrent_RaceCond.png]]

Como se puede ver, la aplicación pasa por un subestado temporal en el que es posible aplicar ambos descuentos, antes de que se marque como canjeado el código.

Hay muchas variantes de este tipo de ataque, incluyendo:

- Canjear una tarjeta de regalo múltiples veces.
- Valorar un producto múltiples veces.
- Transferir más dinero que el que tiene la cuenta.
- Reusar una misma solución CAPTCHA.
- Bypassear un anti-brute-force rate limit.

Los limit overrun son un subtipo de las vulnerabilidades *"time-of-check to time-of-use" (TOCTOU)*.

## Detecting and exploiting limit overrun

Detectar y explotar limit overruns es relativamente sencillo.

1. Identificar un endpoint de un solo uso o rate-limited que tiene algún tipo de impacto en la seguridad o algún otro propósito útil.
2. Lanzar múltiples peticiones al endpoint rápidamente para ver si se puede sobrepasar el límite.

La dificultad está en que la ventana de colisión es muy pequeña (de milisegundos o menos), y aunque se manden las peticiones casi al mismo tiempo, hay factores externos no controlables ni predecibles que afectan a cómo el servidor procesa cada petición y en qué orden.

![[Jitter-async_RaceCond.png]]

### With Burp Repeater

Para solucionar esos problemas, Burp Repeater incorpora técnicas para minimizar el impacto del jitter:

- Para HTTP/1, usa la técnica de **sincronización del último byte**. Va enviando todas las peticiones menos el último byte; cuando todas estén listas, envía el último byte de cada una. El servidor lo procesará como que todas han llegado casi al mismo tiempo.
- Para HTTP/2, usa la técnica del **single-packet attack**. Envía todas las peticiones (20-30) en un solo paquete TCP simultáneamente.

El single-packet attack neutraliza completamente la interferencia del network jitter.

![[Jitter-sync_RaceCond.png]]

**Para llevarlo a cabo**:

1. Se agrupan las peticiones en una carpeta en el Repeater.
2. Se selecciona la opción de mandar las peticiones en paralelo.
3. Se mandan las peticiones.

### With Turbo Intruder

Turbo Intruder permite realizar el single-packet attack, a través de su plantilla `race-single-packet-attack.py`.

**Para utilizarla**:

1. Hay que asegurarse de que el objetivo soporte HTTP/2.
2. Establecer las opciones de configuración `engine=Engine.BURP2` y `concurrentConnections=1` del request engine.
3. Para encolar las peticiones, hay que asignarles un nombre de puerta con el atributo `gate` para el método `engine.queue()`.
4. Para enviar las peticiones, hay que abrir la respectiva puerta con `engine.openGate()`.

``` python
def queueRequests(target, wordlists):
    engine = RequestEngine(endpoint=target.endpoint,
                            concurrentConnections=1,
                            engine=Engine.BURP2
                            )
    
    # queue 20 requests in gate '1'
    for i in range(20):
        engine.queue(target.req, gate='1')
    
    # send all requests in gate '1' in parallel
    engine.openGate('1')
```

-------
# Methodology

Para detectar las secuencias multipasos ocultas, se puede utilizar esta metodología:

## 1. Predict potential collisions

Hacer las siguientes preguntas:

- *¿Este endpoint es crítico?* Muchos endpoints no tocan funcionalidades críticas, por lo que no son relevantes.
- *¿Hay una potencial colisión?* Para una colisión exitosa, se necesitan típicamente dos peticiones o más que desencadenen operaciones sobre el mismo registro. Por ejemplo,, considerando las dos variaciones de implementaciones de password reset:

![[Collitions_RaceCond.png]]

En el primer ejemplo, mandar dos peticiones paralelas no causará una colisión porque cambian registros diferentes. Sin embargo, en el segundo caso, permite editar el mismo registro.

## 2. Probe for clues

Primero hay que ver cómo se comporta el endpoint en condiciones normales. Para ello, *mandar las peticiones de manera secuencial en diferentes conexiones*.

Después, *mandar las peticiones en paralelo*.

Finalmente, buscar algún cambio en el comportamiento en relación a la primera prueba realizada. Puede ser en las respuestas, o en el contenido visible de la aplicación, o el comportamiento en sí.

## 3. Prove the concept

Intentar entender qué está ocurriendo por detrás, eliminar peticiones innecesarias e intentar replicar los efectos.

Tratar la race condition como una debilidad estructural que puede llegar a ser combinada con otra vulnerabilidad para mayor impacto.

--------
# Multi-endpoint race conditions

## Aligning multi-endpoint race windows

Alinear las ventanas de condición de carrera para varios endpoints puede traer varios problemas, incluso cuando se envían las peticiones en un mismo paquete TCP.

![[Align-Multi-endpoint_RaceCond.png]]

Las causas de esos problemas son:

- *Retrasos introducidos por la arquitectura de la red* - Por ejemplo, retrasos al establecer conexiones entre el front-end y el back-end.
- *Retrasos introducidos por el procesamiento específico del endpoint* - Los endpoints varían en el tiempo de procesamiento por las operaciones que realizan.

Las siguientes técnicas permiten potencialmente solucionarlos:

### Connection warming

La conexión back-end típicamente retrasa las peticiones paralelas igualmente.

Para distinguir entre este retraso y el producido por factores específicos del endpoint *se puede enviar una petición o más, por ejemplo a la homepage, para calentar la conexión*, y después mandar el resto sobre la misma conexión.

Si la primera petición tiene un tiempo de procesamiento más largo pero el resto ahora tienen un tiempo más corto de procesamiento, se confirma que el retraso del back-end no está interfiriendo y se puede continuar normal.

Si siguen apareciendo tiempos de respuesta inconsistentes a un endpoint concreto, quiere decir que el retraso del back-end sí está interfiriendo. Es posible que se pueda enfocar usando Turbo Intruder y mandando varias peticiones de calentamiento de conexión antes de mandar las peticiones de ataque.

### Abusing rate or resource limit

Si el calentamiento de conexión no tiene efecto, aún hay varias soluciones para el problema.

Usando Turbo Intruder, se puede *introducir un retraso del lado del cliente*. Sin embargo, esto implica separar las peticiones en distintos paquetes TCP. En objetivos con alto jitter, el ataque seguramente no funcione, independientemente del retraso que se utilice.

![[high-jitter_RaceCond.png]]

En vez de eso, se puede aprovechar una funcionalidad de seguridad común de los servidores web, que consiste en introducir un retraso a las peticiones si se reciben muchas en poco tiempo.

Entonces, enviando muchas peticiones para provocar un rate o resource limit, puede desencadenar en un delay adecuado del lado del servidor.

![[rate-limit-delay_RaceCond.png]]

---------
# Single-endpoint race conditions

Confirmaciones de email o cualquier operación basada en email de un solo endpoint es un buen objetivo para condiciones de carrera de un solo endpoint, ya que los servidores suelen emitir los emails con otro hilo de ejecución.

Como ejemplo, considerando un mecanismo de cambio de contraseña que guarda el ID de usuario y el reset token en la sesión de usuario.

Mandar dos peticiones paralelas sobre la misma sesión, pero con diferentes usuarios, puede causar la siguiente colisión:

![[single-endpoint-collision_RaceCond.png]]

El estado final de las operaciones entonces es el siguiente:

- `session['reset-user'] = victim`
- `session['reset-token'] = 1234`

La sesión ahora contiene el ID de usuario de la víctima, pero el reset token válido se envía al atacante.

*\*Nota*: Puede requerir varios intentos para reproducir la secuencia de operaciones deseada.

---------
# Session-based locking mechanisms

Algunos frameworks como *PHP solo procesan una petición a la vez por sesión*.

Esto puede enmascarar vulnerabilidades explotables. Si se sospecha que las peticiones se están mandando secuencialmente, *probar a mandarlas usando diferentes tokens de sesión*.

--------
# Partial construction race conditions

Muchas aplicaciones crean objetos en múltiples pasos. Por ejemplo, para registrar un nuevo usuario, pueden crear el usuario en la base de datos y luego establecer la API key usando dos sentencias SQL separadas.

Esto puede ser explotable si se inyecta un input que devuelva algo que coincida con el valor no inicializado de la base de datos, como un string vacío, `null` en JSON, y además es parte de un control de seguridad.

En PHP:

- `param[]=foo` <==> `param = ['foo']`
- `param[]=foo&param[]=bar` <==> `param = ['foo', 'bar']`
- `param[]` <==> `param = []`

Para el caso de la API key de antes, esto significa que durante la race window, se pueden realizar peticiones API autenticadas:

``` http
GET /api/user/info?user=victim&api-key[]= HTTP/2
Host: vulnerable-website.com
```

--------
# Time-sensitive attacks

Aunque no se encuentren race conditions, con las mismas técnicas pueden llegar a encontrarse vulnerabilidades que dependan de sincronización precisa.

Un ejemplo es un mecanismo de password reset token que genera el token en base al timestamp (tiempo medido en el instante actual). Entonces, si se mandan dos peticiones de password reset sobre usuarios distintos a la vez, tendrán el mismo token.