
---------
# Definición

>**Information disclosure** (divulgación de información) es una **vulnerabilidad o riesgo de seguridad** que ocurre cuando un sistema, aplicación u organización **revela información sensible a personas no autorizadas**, ya sea de forma accidental o intencional.

Esta información puede incluir, por ejemplo:

- Datos personales (nombres, direcciones, números de identificación)
    
- Credenciales (usuarios, contraseñas, tokens)
    
- Información confidencial de la empresa
    
- Detalles internos del sistema (versiones de software, rutas, errores)
    

En ciberseguridad, el _information disclosure_ es peligroso porque **facilita otros ataques**, como suplantación de identidad, ingeniería social o explotación de vulnerabilidades más graves.

--------
# Common sources of information disclosure

## Files for web crawlers

Archivos como `/robots.txt` o `/sitemap.xml`, destinados a controlar la navegación de los crawlers por su página, pueden revelar directorios que pueden contener información sensible.

## Directory listings

Algunos servidores pueden estar configurados para que muestren la lista de archivos que contiene el directorio si este no tiene presente un index page.

## Developer comments

Algunos comentarios introducidos en el html en la etapa de desarrollo puede que hayan sido omitidos y permanezcan aún en producción. Estos comentarios no son visibles en el renderizado, pero pueden verse simplemente añadiendo `view-source:` antes de la URL, o simplemente haciendo `Ctrl + U`.

## Error messages

Uno de los casos más comunes de divulgación de información son los errores de mensaje detallados. Revelan información sobre el tipo de dato esperado, facilitando el enfoque de la explotación.

En otros casos, puede revelar información sobre qué tecnologías se están utilizando. Pueden mencionar un motor de plantillas, un tipo de base de datos, o el servidor que la web utiliza, junto con la versión. Esta información es útil para encontrar *exploits documentados* en internet, si existe, para esa versión concreta. Además, es útil para buscar en la documentación por *errores comunes en la configuración* o *ajustes por defecto peligrosos* que puedan ser explotables.

La información revelada puede llevar a una tecnología ``open-source``, lo que puede resultar muy valioso para construir un exploit.

También, las diferencias en los mensajes de error se pueden aprovechar para revelar información aún más útil. Observar estas diferencias es una técnica utilizada para explotar [[SQL Injection]], [[Authentication Vulnerabilities#Username enumeration]], entre otras.

## Debugging data

Por motivos de debugging, muchas aplicaciones web generan mensajes de error personalizados y logs que contienen grandes cantidades de información sobre el comportamiento de la aplicación en sí. Esta información es útil en desarrollo, pero también es extremadamente útil para un atacante que se filtre en el entorno de producción.

Los mensajes de debug pueden contener información como:

- Valores de variables de clave de session (*key session variables*) que pueden ser manipulados via input del usuario.
- **Hostnames** y **credenciales** de componentes del back-end.
- **Nombres de archivos y directorios** del servidor.
- **Claves de encriptación** de datos transmitidos por el cliente.

La información de debugging puede estar almacenada en un archivo independiente. Si el atacante consigue acceso a este archivo, puede ser útil para entender el estado de ejecución de la aplicación, así como proporcionar ideas de cómo construir un input que manipule ese estado y controlar la información recibida.

## User account pages

El perfil de usuario o página de la cuenta contienen información sensible, como el correo del usuario, número de teléfono, API key...

Un usuario normalmente solo tiene acceso a su propia página de cuenta. No obstante, algunas páginas web contienen fallas lógicas que permiten a un atacante aprovechar estas páginas para ver datos de otros usuarios. Esta vulnerabilidad se conoce como **IDOR** y está cubierta en: [[Access Control Vulnerabilities#Insecure Direct Object References (IDORs)]].

## Source code disclosure via backup files

Obtener el código fuente del servidor permite conocer el funcionamiento de la aplicación para posteriormente llevar a cabo ataques de gran impacto, o incluso revelar información sensible hard-codeada, como API keys o credenciales de acceso a componentes del back-end.

En ocasiones, es posible que la página web exponga su propio código fuente, aunque usualmente solicitar estos recursos no revela el código en sí. Por ejemplo, archivos `.php` ejecutarán normalmente el código, en vez de mostrar el contenido en texto plano.

A pesar de eso, se puede tratar de conseguir acceso a esa información con diferentes trucos. Por ejemplo, algunos editores de texto generan un archivo de backup temporal mientras el original está siendo editado, y se identifican por tener el caracter `~` al final.

`source.php` <- Archivo original
`source.php~` <- Backup temporal

También es interesante probar por extensiones de archivo típicas de backups, como `.bak`.

## Insecure configuration

Es común encontrar aplicaciones web con configuraciones inseguras debido al gran uso de tecnologías de terceros, de cuyas amplias configuraciones no se tienen conocimientos suficientes para implementarse por parte del desarrollador.

Otro caso son las opciones de debug en el entorno de producción. Como el método `TRACE` de HTTP, que funciona como un echo. Obtienes en la respuesta la petición que le ha llegado al back-end. Este método puede revelar cabeceras internas que hayan sido incluidas por los proxies inversos.

## Version control history

Prácticamente todas las aplicaciones web han sido desarrolladas utilizando un sistema de control de versiones. El más común es Git.

Los proyectos de Git guardan por defecto todos los datos de control de versión en el directorio `.git`. En ocasiones, este directorio está expuesto en producción, siendo posible acceder a él de la forma `/.git`.

La forma práctica de navegar por la información de este directorio es descargarlo en local, y utilizar la propia instalación de git para explorar el contenido. Esto no tiene por qué dar acceso al código fuente como tal, pero sí que permite explorar los logs que contienen los cambias commiteados, entre otra información interesante.