
--------
# IMPORTANTE (temporal)

Para buscar delimitadores válidos para el caso de cache server normalization, es necesario probar los delimitadores solo con el recurso dinámico, porque si se hace con la URL del exploit, la caché mostrará siempre el mismo resultado, ya que la clave será ``/resources``, debido al path traversal.

- MAL -> `http://victim.com/my-account;%2f%2e%2e%2fresources`
- BIEN -> `http://victim.com/my-account;`

Una vez obtenidos los delimitadores aceptados, probar uno por uno en el payload, teniendo en cuenta que la caché no haga hit con el intento anterior.