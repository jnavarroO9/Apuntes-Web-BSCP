
-------
# Definición

>Una **file upload vulnerability** se produce cuando un sistema acepta archivos proporcionados por el usuario y **no aplica controles adecuados de seguridad**. Esto puede permitir que un atacante suba archivos que luego se ejecuten en el servidor o se utilicen para otros ataques.

Estas vulnerabilidades suelen aparecer en funciones como:

- Formularios para subir imágenes de perfil
    
- Carga de documentos (PDF, Word, etc.)
    
- Sistemas de gestión de contenido
    
- Adjuntos en plataformas web
    

Si el sistema no valida correctamente los archivos, el atacante puede aprovecharlo.

## Riesgos principales

Algunas consecuencias comunes son:

- **Ejecución remota de código (RCE)** si el servidor ejecuta archivos subidos (por ejemplo `.php`, `.jsp`, `.aspx`)
    
- **Web shells** que permiten controlar el servidor
    
- **Defacement** del sitio web
    
- **Subida de malware**
    
- **Denegación de servicio (DoS)** mediante archivos grandes o maliciosos
    
- **Exposición de información sensible**

## Problemas comunes que causan esta vulnerabilidad

1. **Validación solo por extensión**
    
2. **Falta de verificación del tipo real del archivo**
    
3. **Guardar archivos en directorios ejecutables**
    
4. **No renombrar archivos subidos**
    
5. **No limitar tamaño o tipo de archivo**
    
6. **Procesamiento inseguro de archivos (ej. ImageMagick exploits)**
    

## Medidas de mitigación

Buenas prácticas para prevenir estas vulnerabilidades:

- **Lista blanca de extensiones permitidas** (ej. solo `.jpg`, `.png`, `.pdf`)
    
- **Validar el tipo MIME y el contenido real**
    
- **Renombrar archivos subidos** con nombres aleatorios
    
- **Guardar archivos fuera del directorio público**
    
- **Deshabilitar ejecución en directorios de subida**
    
- **Limitar tamaño del archivo**
    
- **Escanear archivos con antivirus**
    
- **Usar Content Security Policy y controles adicionales**

--------
# Vías de explotación

>Todas las técnicas están enfocadas en servidores que soportan archivos `.php`.

## Remote code execution via web shell upload

El método para ejecutar comandos en el servidor a través de una funcionalidad de subida de archivos se basa en subir un archivo ejecutable, con el siguiente contenido:

![[Web_Shell_File_Upload.png]]

Para ejecutar comandos, hay que navegar hasta la ubicación de la web shell en el servidor, por ejemplo `/my-account/avatar/shell.php`, y introducir los comandos en el parámetro `cmd` a través del método `GET`.

- `http://victim.com/my-account/avatar/shell.php?cmd=whoami`

*\*Nota*: El objetivo final de una explotación de subida de archivos es subir una web shell como la anterior. Todas las técnicas siguientes se centran en cómo eludir defensas contra este tipo de ataque.

## Web shell upload via Content-Type restriction bypass

Si la aplicación confía únicamente en la cabecera `Content-Type` para comprobar el tipo de archivo subido, simplemente se puede cambiar el valor de esta por uno permitido, como `image/png` para una funcionalidad de subida de imágenes.

![[Content-Type_File_Upload.png]]

## Web shell upload via path traversal

Cuando la ejecución de archivos está bloqueada en el directorio donde se ubican los archivos subidos, por ejemplo a través de un `.htaccess`, se puede tratar de realizar un path traversal para situar el archivo en otro directorio donde sí se pueda ejecutar.

![[Path_Traversal_File_Upload.png]]

- Es posible que se necesite url-encodear la secuencia `../`.
- Es necesario que el archivo se ubique en un directorio accesible como cliente.

## Web shell upload via extension blacklist bypass

Cuando la aplicación bloquea extensiones, es muy difícil que abarque todas las posibles, incluso las variantes de una misma. Hay varias aproximaciones:

### 1. HackTricks

La primera es seguir las técnicas de este [artículo de hacktricks](https://book.hacktricks.wiki/en/pentesting-web/file-upload/index.html).

### 2. .htaccess

Subir un archivo `.htaccess` que permita una extensión custom en el servidor y haga que este la trate como si fuera de un archivo php. O directamente subir htshells de internet.

``` php
AddType application/x-httpd-php .josmo
```

Después, subir la web shell como: `shell.josmo`.

## Web shell upload via obfuscated file extension

Sucede cuando el servidor valida la extensión del nombre del archivo. Se pueden probar métodos de hacktricks, o también introducir un *null byte* separando extensiones y provocando discrepancias entre la forma de validar y la forma de guardar el archivo.

*\*Nota*: Para PHP, la técnica del null byte solo afecta versiones antiguas.

``` text
filename="shell.php%00.jpg"

Valida extensión: .jpg -> Aceptado
Guarda archivo:   shell.php
```

## Remote code execution via polyglot web shell upload

Si la aplicación valida que los *magic numbers*, o sea, la secuencia de caracteres que se encuentra al principio del archivo e identifica el tipo del mismo, correspondan con los del archivo esperado, y no uno malicioso como php. Sin embargo, falla al validar si contiene código malicioso en él.

Entonces, se puede partir de una imagen normal, e inyectar código php en los metadatos de la imagen, por ejemplo en el campo ``Comment``. Esto no altera la imagen y puede seguir visualizándose; sin embargo, si se logra subir este archivo políglota con una extensión que el servidor ejecute como php, se va a poder ejecutar comandos.

*\*Nota*: Con esta aproximación es probable que no funcione bien el comando pasado por `GET`. Es por eso que en el ejemplo se incluye el comando directamente en los metadatos.

![[Polyglot_File_Upload.png]]

## Web shell upload via race condition

Sucede cuando el servidor guarda el archivo subido temporalmente para hacer comprobaciones, y este archivo temporal es accesible y tiene un nombre predecible.

Aunque la existencia de este archivo en el servidor es ínfima, si se mandan las peticiones en el momento oportuno se puede llegar a acceder al archivo durante ese instante.

El payload siguiente de TurboIntruder manda una petición de `POST` para subir la web shell, junto con 5 peticiones `GET` para acceder a ella:

``` python
def queueRequests(target, wordlists):
    engine = RequestEngine(endpoint=target.endpoint, concurrentConnections=10,)

    request1 = '''<YOUR-POST-REQUEST>'''

    request2 = '''<YOUR-GET-REQUEST>'''

    # the 'gate' argument blocks the final byte of each request until openGate is invoked
    engine.queue(request1, gate='race1')
    for x in range(5):
        engine.queue(request2, gate='race1')

    # wait until every 'race1' tagged request is ready
    # then send the final byte of each request
    # (this method is non-blocking, just like queue)
    engine.openGate('race1')

    engine.complete(timeout=60)


def handleResponse(req, interesting):
    table.add(req)
```

