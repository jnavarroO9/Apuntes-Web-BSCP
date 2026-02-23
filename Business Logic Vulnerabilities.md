
--------
# Definición

>Una **business logic vulnerability** es una debilidad que permite a un atacante **manipular flujos, procesos o reglas del negocio** para obtener beneficios indebidos (dinero, acceso, ventajas) aprovechando **suposiciones incorrectas** hechas por los desarrolladores sobre el comportamiento del usuario.

## Ejemplos comunes

- Comprar un producto con **precio negativo o cero**
    
- Aplicar **el mismo cupón de descuento múltiples veces**
    
- Saltarse pasos obligatorios (p. ej., **pagar sin completar el checkout**)
    
- Transferir dinero **sin verificar límites o saldos**
    
- Cambiar el **orden de las acciones** para obtener un resultado no permitido
    
- Repetir una acción que debía ejecutarse **solo una vez** (race conditions lógicas)

## Por qué son peligrosas

- No suelen ser detectadas por scanners automáticos
    
- Requieren **entender el negocio**, no solo el código
    
- Pueden causar **pérdidas financieras directas**
    
- A menudo pasan code reviews porque “todo funciona”

-----------
# Ejemplos de vulnerabilidades de lógica de negocio

Las vulnerabilidades de la lógica de negocio son específicas del contexto de la aplicación, luego su aparición y explotación difiere entre las diferentes aplicaciones web.

En esta sección se comentan algunos ejemplos de errores típicos que los equipos de diseño y desarrollo cometen y que llevan directamente a fallas de la lógica de negocio.

## Excessive trust in client-side controls

Una falla fundamental es asumir que los usuarios van a interactuar con la aplicación a través de la interfaz web proporcionada. Esto es peligroso porque lleva también a asumir que la validación del lado del cliente previene a los usuarios de introducir inputs maliciosos.

Sin embargo, los atacantes pueden utilizar herramientas como Burpsuite, que permiten manipular los datos después de ser mandados por el navegador, pero antes de ser pasados a la lógica del lado del servidor.

## Failing to handle unconventional input

Una aplicación puede estar diseñada para aceptar valores arbitrarios de un tipo de datos específico, y es la lógica la que determina si ese valor es aceptable o no desde la perspectiva del negocio. Si no hay una lógica explícita para tratar un caso concreto, esto puede llevar a comportamientos inesperados y potencialmente explotables.

Por ejemplo, considerando una transferencia entre dos bancos, la funcionalidad siguiente comprueba que la entidad que la va a realizar tenga suficientes fondos para ella:

``` php
$transferAmount = $_POST['amount'];
$currentBalance = $user->getBalance();

if ($transferAmount <= $currentBalance) {
    // Complete the transfer
} else {
    // Block the transfer: insufficient funds
}
```

No obstante, esta lógica no previene a los usuarios de proporcionar valores negativos en el parámetro `amount`. Esto puede llevar, por ejemplo, dependiendo de como funcione el resto de la lógica, a que la transferencia se realice en el sentido contrario, siendo el emisor el que reciba el dinero.

Estas fallas lógicas son fáciles de pasar por alto tanto en el desarrollo como en el testing, especialmente cuando inputs de este estilo pueden estar siendo bloqueados por los controles de la interfaz web.

Las claves para encontrar este tipo de vulnerabilidades es preguntarse las siguientes cuestiones:

- *¿Hay límites impuestos en los datos enviados?*
- *¿Qué pasa si se alcanzan dichos límites?*
- *¿Se está aplicando alguna transformación o normalización al input introducido?*

## Making flawed assumptions about user behavior

Una de las causas principales de las vulnerabilidades de lógica es suponer erróneamente el comportamiento del usuario. Algunos ejemplos de estos supuestos son:

### Trusted users won't always remain trustworthy

Es el caso en el que la aplicación implementa medidas de protección robustas, pero falla asumiendo que, una vez pasados estos controles, el usuario y sus datos son confiables indefinidamente.

Por ejemplo, teniendo en cuenta una aplicación que tiene un panel de administración en `/admin` y que controla su acceso a usuarios registrados en el dominio de la empresa `@corp.com`. Un usuario cualquiera no puede acceder entonces a este endpoint, y tampoco puede crear un usuario con ese dominio porque require de verificación del email.

Sin embargo, una vez registrado en la página, el usuario puede cambiar su correo por otro cualquiera, siendo posible introducir `user@corp.com` y accediendo así a `/admin`.

### Users won't always supply mandatory input

Suele deberse a asumir que los parámetros de un formulario marcados como *required* van a ser siempre enviados al back-end, pero estos parámetros pueden modificarse con Burpsuite.

Esto es un problema cuando existen múltiples funciones implementadas en el mismo script del servidor, y la presencia o ausencia de un parámetro en particular determina cuál de ellas es ejecutada. Eliminar parámetros puede llevar al atacante a acceder a caminos de ejecución supuestamente fuera de su alcance.

Para probar por estos fallos lógicos hay que tener en cuenta lo siguiente:

- *Solo eliminar un parámetro cada vez*, para asegurar que todos los caminos relevantes se han alcanzado.
- *Intentar eliminar tanto el valor del parámetro como también el nombre*. El servidor típicamente tratará los dos casos de manera diferente.
- *Seguir los procesos multi-etapa hasta que se completan*. A veces, modificar un parámetro en una etapa tiene efecto en una etapa posterior en el workflow.

Esto aplica tanto a parámetros URL como `POST`, pero también es importante comprobar las **cookies**.

### Users won't always follow the intended sequence

Muchas transacciones se basan en un workflow predefinido que consiste en una secuencia de pasos. La interfaz web usualmente guía a los usuarios a través de este proceso; sin embargo, un atacante no necesariamente sigue la secuencia intencionada.

El proceso para testear vulnerabilidades del workflow se basa en probar combinaciones de etapas, usualmente a través de peticiones `GET` y `POST`, a diferentes endpoints o incluso al mismo. Alternar el orden de pasos, o incluso repetir algunos, pueden llevar a saltar ciertos controles por parte del servidor, o realizar acciones que no se esperaban.

Este tipo de pruebas suelen causar excepciones por variables con valores *null* o no inicializadas, además de alcanzar estados inconsistentes que muestren errores. Estos mensajes de error pueden ser de gran valor si revelan información de debug o directamente del servidor.

## Domain-specific flaws

En muchas ocasiones se pueden encontrar fallos lógicos que son específicos del dominio de negocio o del propósito del sitio. Por ejemplo, un vector común de las tiendas online es la funcionalidad de descuentos.

Hay que prestar particular atención a cualquier situación en la que los precios u otro valor sensible es ajustado en base a criterios determinados por acciones de usuario. Intentar comprender que algoritmos se están aplicando y en qué momento, con el objetivo de conseguir un estado de la aplicación en el que los ajustes aplicados no corresponden con los criterios esperados por los desarrolladores.

También es importante tener un conocimiento del dominio específico de la aplicación para poder identificar comportamientos peligrosos de la aplicación que otros testers sin ese conocimiento hayan podido pasar por alto.

-*Ejemplo*:
Una tienda online ofrece un descuento del 10% para pedidos de más de $1000. Esto puede ser vulnerable si la aplicación falla en detectar si el pedido ha cambiado después de aplicar el descuento. En ese caso, un atacante podría añadir al carrito productos hasta alcanzar los $1000, canjear el descuento, y eliminar los productos que no quiere antes de realizar el pedido.

## Providing an encryption oracle

Escenarios peligrosos pueden ocurrir cuando el input controlado por el usuario es encriptado y el texto cifrado se muestra al usuario de alguna manera. Este tipo de inputs se conocen como "*encryption oracle*". Un atacante puede utilizar este input para cifrar datos arbitrarios usando el algoritmo y clave simétrica correctos.

Esto se vuelve peligroso cuando existen otros inputs controlados por el usuario que esperan datos cifrados con el mismo algoritmo. En este caso, el atacante puede potencialmente utilizar el oráculo de encriptación para generar input encriptado válido, y posteriormente pasarlo a otras funciones sensibles.

Este problema puede agravarse si existe otro input que proporciona la función inversa. Esto permitiría al atacante desencriptar otra data para identificar la estructura esperada. Esto ahorra mucho tiempo, pero no es estrictamente necesario para construir un exploit.

La severidad de un oráculo de encriptación depende de la funcionalidad que emplee el mismo algoritmo como oráculo.

## Email address parser discrepancies

El parsing de emails es realmente complejo, y es por eso que pueden existir discrepancias en cómo tratan las direcciones email diferentes componentes de la aplicación.

Un atacante puede explotar estas discrepancias usando técnicas de codificación, de modo que pase las validaciones iniciales pero sea interpretado de manera diferente por la lógica de parsing del servidor.

El impacto principal de la discrepancia de parsing de emails es acceso no autorizado a funcionalidades de la aplicación.

Para detalles de explotación, ver el artículo: [Splitting the Email Atom: Exploiting Parsers to Bypass Access Controls](https://portswigger.net/research/splitting-the-email-atom)