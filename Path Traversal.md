
-------------
# Definición

>**Path Traversal** (también conocido como _Directory Traversal_) es una vulnerabilidad de seguridad en aplicaciones web o sistemas que ocurre cuando un atacante manipula las rutas de archivos que la aplicación recibe como entrada, con el fin de acceder a directorios o archivos fuera de la ubicación prevista por el desarrollador.

## Ejemplo típico

Un sistema espera un archivo como parámetro:

``` URL
http://sitio.com/ver?archivo=report.pdf
```

Un atacante podría modificarlo a:

``` URL
http://sitio.com/ver?archivo=../../etc/passwd
```

Si no existe un control adecuado, la aplicación terminará mostrando el archivo del sistema `/etc/passwd` (en Linux), exponiendo información crítica.

## Consecuencias

- **Lectura** de archivos sensibles (configuración, contraseñas, llaves privadas).
    
- **Ejecución** de código malicioso si se combina con otras vulnerabilidades.
    
- **Escalada** de privilegios o acceso no autorizado a datos de usuarios.
    

## Prevención

- **Normalizar** y **validar** las rutas recibidas.
    
- **Restringir** el acceso a un directorio específico (_chroot jail_, sandbox).
    
- **Evitar** concatenar rutas directamente con entradas del usuario.
    
- Usar **funciones seguras** para manejar archivos.

--------
# Explotación básica

La web carga una archivo, en este caso imágenes, de la siguiente forma:

``` URL
http://victim.com/load?filename=image.jpg
```

Por detrás, lo que está ocurriendo es que se está concatenando la ruta del directorio donde se almacenan las imágenes del servidor `/var/www/images/`, con el nombre de archivo de la variable *filename*, resultando en:

- `/var/www/images/image.jpg`

Si no hay sanitización del input, se puede tratar de retroceder directorios hasta llegar a la raíz. Una vez ahí, ya se puede referenciar cualquier archivo del servidor, provocando un LFI.

``` URL
http://victim.com/load?filename=../../../etc/passwd
```

----------
# Absolute path bypass

En ocasiones en las que el servidor bloquee completamente las secuencias de **../**, puede que se de el caso en el que una ruta de archivo absoluto se interprete como tal, en vez de interpretarse como una ruta relativa. De esta manera se puede bypassear la restricción y conseguir LFI.

``` URL
http://victim.com/load?filename=/etc/passwd
```

---------
# Sequences stripped non-recursively

Si la cadena *../* se está eliminando al pasar por un filtro de manera no recursiva, se puede introducir:

``` URL
http://victim.com/load?filename=....//....//....//etc/passwd
```

Al pasar por el filtro, queda la siguiente cadena:

``` URL
http://victim.com/load?filename=../../../etc/passwd
```

------------
# Sequences stripped with superfluous URL-decode

En este caso, el servidor realiza una decodificación de la URL extra. Además, valida por secuencias ilegales antes de esa decodificación. Luego se puede explotar un path traversal con un **doble URL-encode**.

``` URL
http://victim.com/load?filename=..%252f..%252f..%252fetc/passwd
```

- En ASCII:
	- */* --> *%2f*
	- *%* --> *%25*

-----------
# Validation of start of path

Si la referencia al archivo es mediante una ruta absoluta, y el servidor valida únicamente que el inicio del path coincida con la ruta del directorio donde se almacenan los recursos de la web, se puede bypassear con la misma técnica de retroceso de directorios.

``` URL
http://victim.com/load?filename=/var/www/images/../../../etc/passwd
```

---------
# Validation of file extension with null byte bypass

Si se valida por la extensión del archivo, algo que se puede probar es a realizar una inyección de null byte. Consiste en introducir un *%00* antes de la extensión de archivo para que pase la validación de extensión, pero a la hora de incluir el archivo, busque en el sistema por el nombre hasta el null byte, sin incluir la extensión.

*\*Nota*: Esto suele afectar solamente a aplicaciones desactualizadas (php <5.3.8) o aplicaciones independientes mal diseñadas.

``` URL
http://victim.com/load?filename=/etc/passwd%00.jpg
```