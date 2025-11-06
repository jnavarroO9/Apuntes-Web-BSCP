
---------
# Definición

>**Server-Side Template Injection (SSTI)** es una vulnerabilidad que ocurre cuando una aplicación web inserta datos no confiables dentro de una plantilla del servidor (template) sin validarlos ni escaparlos correctamente, permitiendo que un atacante injecte código de plantilla que el motor de plantillas ejecute en el servidor.

## Riesgos principales

- Exposición de información sensible (archivos, claves, variables de entorno).
    
- Ejecución remota de código en el servidor (en casos graves).
    
- Escalada a compromiso completo de la aplicación/infraestructura.

## Mitigaciones recomendadas

1. **No permitir** que entradas del usuario lleguen como código a la plantilla.
    
2. **Escapar/filtrar** adecuadamente las variables cuando se rendericen.
    
3. **Usar APIs seguras** del motor que separen datos de la plantilla (p. ej. pasar solo valores, no plantillas completas).
    
4. **Validación y normalización** de entradas (whitelisting cuando sea posible).
    
5. **Principio de menor privilegio**: que el proceso de la app tenga acceso mínimo a archivos y secretos.
    
6. **Actualizaciones y parches** del motor de plantillas y librerías relacionadas.
    
7. **Revisión de código** y pruebas de seguridad periódicas (SAST/DAST).

-------------
# Explotación básica

La metodología para explotar un SSTI consiste en *reconocer que template* se está empleando en la página web, y *localizar un campo* que se refleje directamente en la web y que podría estar pasándose al renderizador de la plantilla.

Al conocer el template, se conoce la sintaxis de etiquetas, luego se prueba a inyectar una operación aritmética. Si en la respuesta se obtiene el resultado de la operación, entonces existe un SSTI.

\*Ejemplo en Ruby/ERB:

- `<%= 7*7 %>` --> `49`

En la documentación de la plantilla o en páginas como [PayloadsAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings/) o [Hacktricks](https://book.hacktricks.wiki/en/index.html) se pueden encontrar formas de realizar RCE.

---------
# Code context

Si el parámetro controlado *se introduce en un template expression*, y después se pasa al motor de plantillas para renderizarse, se puede tratar de *escapar del contexto* y crear otra expresión de plantilla para lograr la ejecución remota de comandos.

\*Ejemplo en Python/Tornado:

- El nombre de usuario que se muestra en la sección de comentarios se puede elegir. Al cambiarlo, se introduce este en una expresión de template.

	``{{user.nickname}}``

- Escapando del contexto, podemos ejecutar comandos:

	-Texto introducido: `user.nickname}}{%import os%}{{os.system('whoami')`

	-Resultado en el código: ``{{user.nickname}}{%import os%}{{os.system('whoami')}}``

----------
# Using documentation

## Engine identification

Para identificar que motor se está empleando, se puede tratar de *forzar un error*, como modificar un atributo de una variable que se está cargando en una expresión de plantilla. Esto leakeará el nombre del motor.

## Exploiting RCE

Una vez identificado el motor, mediante la documentación o con las herramientas mencionadas en [[#Explotación básica]] se puede llevar a cabo la ejecución de comandos.

---------
# Information disclosure via user-supplied objects

En ocasiones, se pude conseguir información sensible en base a atributos de objetos internos que se pueden llamar desde las expresiones de plantilla.

Por ejemplo, en Django, si se introduce la expresión ``{{% debug %}}``, se logran ver objetos a los que se puede hacer referencia. Si en el output uno de los objetos que aparecen es *settings*, se puede leakear la clave secreta del framework con:

- `{{ settings.SECRET_KEY }}`

----------
# SSTI in sandboxed environments

Una medida de seguridad que pueden presentar los motores de plantilla es que el contenido de las expresiones de plantilla se ejecuta en un *entorno aislado* al del código de la aplicación web. No obstante, hay veces en las que se puede escapar de este entorno y realizar acciones no esperadas.

\*Ejemplo en Java/Freemarker:

En versiones antiguas de freemarker, se podía realizar un LFI aprovechando una secuencia de métodos aplicados sobre el objeto product:

- `${product.getClass().getProtectionDomain().getCodeSource().getLocation().toURI().resolve('path_to_the_file').toURL().openStream().readAllBytes()?join(" ")}`

-------
# SSTI with custom exploit

Si la inyección de expresiones de plantilla se ve limitada por un *sandbox*, se puede intentar ejecutar ciertas acciones privilegiadas dependiendo del contexto de la página web.

- Utilizando las *funcionalidades* de la web y forzando errores se pueden llegar a leakear *funciones y objetos internos* del backend.
- Estos objetos y funciones pueden llamarse desde el contexto del sandbox.

El alcance de esta aproximación varía en función de los métodos y objetos que hayan disponibles. Comúnmente se suele lograr LFI.

