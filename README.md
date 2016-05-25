# Feing facilita crear clientes http usando Java

[![Join the chat at https://gitter.im/Netflix/feign](https://badges.gitter.im/Join%20Chat.svg)](https://gitter.im/Netflix/feign?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)
Feign es una envoltura de invocaciones http a java inspirado por [Retrofit](https://github.com/square/retrofit), [JAXRS-2.0](https://jax-rs-spec.java.net/nonav/2.0/apidocs/index.html), y [WebSocket](http://www.oracle.com/technetwork/articles/java/jsr356-1937161.html).  El primer objetivo de Feign fue reducir la complejidad de envolver [Denominator](https://github.com/Netflix/Denominator) de manera uniforme a los apis http sin importar el estilo [restfulness](http://www.slideshare.net/adrianfcole/99problems).

### Por qué Feign y no X?

Puedes usar herramientas cono Jersey y CXF para escribir clientes Java para servicios SOAP o REST. Puedes escribir tu propio codigo usando librerias de transporte http como Apache HC. Feign ayuda a conectar tu codigo con apis http con una sobrecarga minima de procesamiento y codigo. Usando decoders personalizados y manejadores de errores, puedes ser capaz de escribir cualquier api http basado en texto.

### Cómo trabaja Feign?

Feign trabaja procesando anotaciones dentro de request basados en plantillas. Justo antes de enviarlo, los argumentos son aplicados a estas plantillas de manera sencilla. Mientras esto limita a Feign a solo soportar apis basados en texto, simplifica dramaticamente aspectos del sistema tales como permitir reproducir peticiones. Es extremadamente sencillo realizar pruebas unitarias sobre las conversiones conociendo esto.

### Aspectos básicos

El uso normalmente se mira como esto: una adaptacion del uso de [canonical Retrofit sample](https://github.com/square/retrofit/blob/master/samples/src/main/java/com/example/retrofit/SimpleService.java).

```java
interface GitHub {
  @RequestLine("GET /repos/{owner}/{repo}/contributors")
  List<Contributor> contributors(@Param("owner") String owner, @Param("repo") String repo);
}

static class Contributor {
  String login;
  int contributions;
}

public static void main(String... args) {
  GitHub github = Feign.builder()
                       .decoder(new GsonDecoder())
                       .target(GitHub.class, "https://api.github.com");

  // Fetch and print a list of the contributors to this library.
  List<Contributor> contributors = github.contributors("netflix", "feign");
  for (Contributor contributor : contributors) {
    System.out.println(contributor.login + " (" + contributor.contributions + ")");
  }
}
```

### Personalización

Feign tiene varios aspectos que pueden ser personalizados. Para casos simples puedes usar `Feign.builder()` para construir una interface de un API con tus componentes personalizados. Por ejemplo:

```java
interface Bank {
  @RequestLine("POST /account/{id}")
  Account getAccountInfo(@Param("id") String id);
}
...
Bank bank = Feign.builder().decoder(new AccountDecoder()).target(Bank.class, "https://api.examplebank.com");
```

### Interfaces Multiples
Feign puede producir multiples interfaces para un api. Estas son definidas como `Target<T>` (por defecto `HardCodedTarget<T>`), lo cual permite dinamicamente descubrir y decorar las peticiones antes de ser ejecutadas.

Por ejemplo, el siguiente patron podría decorar cada petición con el url actual y el auth token del servicio de identidad.

```java
Feign feign = Feign.builder().build();
CloudDNS cloudDNS = feign.target(new CloudIdentityTarget<CloudDNS>(user, apiKey));
```

### Ejemplos
Feign incluye clientes de ejemplo para [GitHub](./example-github) y [Wikipedia](./example-wikipedia). El proyecto denominator tambien puede ser usado para Feign en la práctica. De manera particular, mira ésto [example daemon](https://github.com/Netflix/denominator/tree/master/example-daemon).

### Integraciones
Feign tiene la intención de realizar un buen trabajao dentro de Netflix y otras comunidades Open Source. Los modulos son bienvenidos para ser integrados con tus proyectos favoritos!

### Gson
[Gson](./gson) incluye un codificador y decodificador que puedes usarlo con un API JSON.

Agrega `GsonEncoder` y/o `GsonDecoder` a tu `Feign.Builder` así:

```java
GsonCodec codec = new GsonCodec();
GitHub github = Feign.builder()
                     .encoder(new GsonEncoder())
                     .decoder(new GsonDecoder())
                     .target(GitHub.class, "https://api.github.com");
```

### Jackson
[Jackson](./jackson) incluye un codificador y decodificador que puedes usarlo con un API JSON.

Agrega `JacksonEncoder` y/o `JacksonDecoder` a tu `Feign.Builder` así:

```java
GitHub github = Feign.builder()
                     .encoder(new JacksonEncoder())
                     .decoder(new JacksonDecoder())
                     .target(GitHub.class, "https://api.github.com");
```

### Sax
[SaxDecoder](./sax) permite decodificar XML de forma que es compatible con la JVM normal y también dentro de ambientes Android.

Aquí un ejemplo de como configurar un análisis sintáctico de un reponse Sax:
```java
api = Feign.builder()
           .decoder(SAXDecoder.builder()
                              .registerContentHandler(UserIdHandler.class)
                              .build())
           .target(Api.class, "https://apihost");
```

### JAXB
[JAXB](./jaxb) incluye un codificador y decodificador que puedes usarlo con un API XML.

Agrega `JAXBEncoder` y/o `JAXBDecoder` a tu `Feign.Builder` así:

```java
api = Feign.builder()
           .encoder(new JAXBEncoder())
           .decoder(new JAXBDecoder())
           .target(Api.class, "https://apihost");
```

### JAX-RS
[JAXRSContract](./jaxrs) sobre-escribe el procesamiento de las anotaciones, para que en su lugar, se haga uso del estandar provisto por la especificación JAX-RS. Actualmente esta orientado a la versión 1.1 spec.

Aquí puedes ver el ejemplo anterior re-escrito para usar JAX-RS:
```java
interface GitHub {
  @GET @Path("/repos/{owner}/{repo}/contributors")
  List<Contributor> contributors(@PathParam("owner") String owner, @PathParam("repo") String repo);
}
```
```java
GitHub github = Feign.builder()
                     .contract(new JAXRSContract())
                     .target(GitHub.class, "https://api.github.com");
```
### OkHttp
[OkHttpClient](./okhttp) direcciona las peticiones http de Feign a [OkHttp](http://square.github.io/okhttp/), lo cual habilita SPDY y un mejor control de la red.

Para usar OkHttp con Feign, agrega el modulo OkHttp a tu classpath. Luego, configura Feign para usar OkHttpClient:

```java
GitHub github = Feign.builder()
                     .client(new OkHttpClient())
                     .target(GitHub.class, "https://api.github.com");
```

### Ribbon
[RibbonClient](./ribbon) sobre-escribe la resolución del URL del cliente de Feign, agrega un ruteo inteligente y capacidades de recuperación provistas por [Ribbon](https://github.com/Netflix/ribbon).

La integración requiere que tu pases el nombre de tu cliente ribbon, como si fuera el host del url, por ejemplo `myAppProd`.
```java
MyService api = Feign.builder().client(RibbonClient.create()).target(MyService.class, "https://myAppProd");

```

### Hystrix
[HystrixFeign](./hystrix) configura soporte para circuit breaker provisto por [Hystrix](https://github.com/Netflix/Hystrix).

Para usar Hystrix con Feign, agrega el modulo Hystrix a tu classpath. Luego usa el builder `HystrixFeign`:

```java
MyService api = HystrixFeign.builder().target(MyService.class, "https://myAppProd");

```

### SLF4J
[SLF4JModule](./slf4j) permite dirigir los logs de Feign hacia [SLF4J](http://www.slf4j.org/), permitiendo facilmente usar el componente de logs de backend que tu escojas (Logback, Log4J, etc.)

Para usar SLF4J con Feign, agrega ambas cosas, el módulo SLF4J y el binding SLF4J de tu elección al classpath. Luego, configura Feign para usar Slf4jLogger:

```java
GitHub github = Feign.builder()
                     .logger(new Slf4jLogger())
                     .target(GitHub.class, "https://api.github.com");
```

### Decodificadores
`Feign.builder()` permite especificar configuración adicional acerca de como decodificar un response.

Si alguno de los métodos en tu interface retornan otros tipos además de `Response`, `String`, `byte[]` or `void`, necesitarás configurar un `Decoder` distinto al que está configurado por defecto.

Aquí puedes ver como configurar una decodificación JSON (usando la extensión `feign-gson`):

```java
GitHub github = Feign.builder()
                     .decoder(new GsonDecoder())
                     .target(GitHub.class, "https://api.github.com");
```

### Codificadores
La forma mas sencilla de enviar el cuerpo de una petición hacia un servidor es definir un método `POST` que tiene un parametro `String` o `byte[]` sin ninguna anotación sobre él. Es probable que necesites agregar una cabecera `Content-Type`.

```java
interface LoginClient {
  @RequestLine("POST /")
  @Headers("Content-Type: application/json")
  void login(String content);
}
...
client.login("{\"user_name\": \"denominator\", \"password\": \"secret\"}");
```

Al configurar un `Encoder`, puedes enviar el cuerpo de la petición asegurandote que no haya errores en él al construirlo. Aquí un ejemplo usando la extensión `feign-gson`:

```java
static class Credentials {
  final String user_name;
  final String password;

  Credentials(String user_name, String password) {
    this.user_name = user_name;
    this.password = password;
  }
}

interface LoginClient {
  @RequestLine("POST /")
  void login(Credentials creds);
}
...
LoginClient client = Feign.builder()
                          .encoder(new GsonEncoder())
                          .target(LoginClient.class, "https://foo.com");

client.login(new Credentials("denominator", "secret"));
```

### @Body templates
La anotación `@Body` indica una plantilla para expandir el uso de parametros anotados con `@Param`. Probablemente necesitarás agregar la cabecera `Content-Type`.

```java
interface LoginClient {

  @RequestLine("POST /")
  @Headers("Content-Type: application/xml")
  @Body("<login \"user_name\"=\"{user_name}\" \"password\"=\"{password}\"/>")
  void xml(@Param("user_name") String user, @Param("password") String password);

  @RequestLine("POST /")
  @Headers("Content-Type: application/json")
  // json curly braces must be escaped!
  @Body("%7B\"user_name\": \"{user_name}\", \"password\": \"{password}\"%7D")
  void json(@Param("user_name") String user, @Param("password") String password);
}
...
client.xml("denominator", "secret"); // <login "user_name"="denominator" "password"="secret"/>
client.json("denominator", "secret"); // {"user_name": "denominator", "password": "secret"}
```

### Headers
Feign soporta configuraciones de cabeceras como parte del api o como parte del cliente dependiendo del caso de uso.

#### Set de headers (cabeceras) usando apis
En casos donde ciertas interfaces o llamadas deban siempre tener ciertos valores de las cabeceras configurados, tiene sentido definir cabeceras como parte del api.

Cabeceras estaticas pueden ser configuradas sobre una interface del api o sobre un metodo usando la anotación `@Headers`.

```java
@Headers("Accept: application/json")
interface BaseApi<V> {
  @Headers("Content-Type: application/json")
  @RequestLine("PUT /api/{key}")
  void put(@Param("key") String, V value);
}
```

Los métodos pueden especificar contenido dinámico para las cabeceras estáticas usando una variable de expansion en `@Headers`.

```java
 @RequestLine("POST /")
 @Headers("X-Ping: {token}")
 void post(@Param("token") String token);
```

En casos donde los nombres y valores de los campos de las cabeceras son dinámicos y el rango de posibles nombres de campos no pueda se conocido a priori y pueda haber variaciones entre las diferentes invocaciones a los metodos en el mismo api/client (por ejemplo: campos de la cabecera personalizados para la metadata "x-amz-meta-\*" o "x-goog-meta-\*"), un parametro Map puede ser anotado con `HeaderMap` para construir una consulta que use el contenido del mapa para construir una consulta que use el contenido del mapa como parámetros de la cabera

```java
 @RequestLine("POST /")
 void post(@HeaderMap Map<String, Object> headerMap);
```

Estos enfoques especifican campos en la cabecera como parte del api y no requieren personalización cuando construyes un cliente Feign.

#### Configurar cabeceras por target
En casos donde las cabeceras deberían diferir por el mismo api que esta basado en distintos endpoints o donde se requiera una personalización per-request, las cabeceras pueden ser configuradas como parte del cliente usando un `RequestInterceptor` o un `Target`.

Para un ejemplo de configuración de cabeceras usando un `RequestInterceptor`, mira la sección `Request Interceptors`.

Las cabeceras pueden ser configuradas como parte de un `Target` personalizado.

```java
  static class DynamicAuthTokenTarget<T> implements Target<T> {
    public DynamicAuthTokenTarget(Class<T> clazz,
                                  UrlAndTokenProvider provider,
                                  ThreadLocal<String> requestIdProvider);
    ...
    @Override
    public Request apply(RequestTemplate input) {
      TokenIdAndPublicURL urlAndToken = provider.get();
      if (input.url().indexOf("http") != 0) {
        input.insert(0, urlAndToken.publicURL);
      }
      input.header("X-Auth-Token", urlAndToken.tokenId);
      input.header("X-Request-ID", requestIdProvider.get());

      return input.request();
    }
  }
  ...
  Bank bank = Feign.builder()
          .target(new DynamicAuthTokenTarget(Bank.class, provider, requestIdProvider));
```

Estos enfoques dependen de los objetos personalizados: `RequestInterceptor` o `Target` que estan siendo configurados
sobre el cliente Feign cuando es construido y puede ser usado como una forma para asignar cabeceras sobre todas las
llamadas al api on por cada cliente. 
Los métodos se ejecutan cuando la llamada al api es realizada sobre el thread que invoca la llamada al api, 
lo cual permite que las cabeceras sean asignadas dinamicamente al momento de la invocación y de alguna manera dentro de 
un contexto específico -- por ejemplo, un almacen thread-local puede ser usado para configurar diferentes valores en la
cabecera dependiendo del thread que realiza la invocación, lo cual puede ser de mucha ayuda para cosas tales como 
configurar rastros de indentificadores especificos por cada thread en las invocaciones.

### Uso avanzado

#### Base Apis
En varios casos, los apis para un servicio siguen las mismas convenciones. Feign soportan este patrón a traves de herencia simple en las interfaces.

Consider the example:
```java
interface BaseAPI {
  @RequestLine("GET /health")
  String health();

  @RequestLine("GET /all")
  List<Entity> all();
}
```

Puedes definir y apuntar hacia un api específico, heredando los metodos base.
```java
interface CustomAPI extends BaseAPI {
  @RequestLine("GET /custom")
  String custom();
}
```

En varios casos, las representaciones de los recursos también son consistentes. Por esta razón, los tipos de parametros son soportados sobre la base de la interface del api.

```java
@Headers("Accept: application/json")
interface BaseApi<V> {

  @RequestLine("GET /api/{key}")
  V get(@Param("key") String);

  @RequestLine("GET /api")
  List<V> list();

  @Headers("Content-Type: application/json")
  @RequestLine("PUT /api/{key}")
  void put(@Param("key") String, V value);
}

interface FooApi extends BaseApi<Foo> { }

interface BarApi extends BaseApi<Bar> { }
```

#### Logging
Puedes logear los mensajes http que van a y desde el target configurando un `Logger`.  Aquí esta la forma mas fácil de hacerlo:
```java
GitHub github = Feign.builder()
                     .decoder(new GsonDecoder())
                     .logger(new Logger.JavaLogger().appendToFile("logs/http.log"))
                     .logLevel(Logger.Level.FULL)
                     .target(GitHub.class, "https://api.github.com");
```

El SLF4JLogger (mira lo anterior) podría también ser de interes.


#### Request Interceptors / Interceptores de peticiones
Cuando necesitas cambiar todos las peticiones, sin importar su target (destino), tu vas a deseas configurar un `RequestInterceptor`.
Por ejemplo, si actuas como intermediario, podrías deseas propagar la cabecera `X-Forwarded-For`.

```java
static class ForwardedForInterceptor implements RequestInterceptor {
  @Override public void apply(RequestTemplate template) {
    template.header("X-Forwarded-For", "origin.host.com");
  }
}
...
Bank bank = Feign.builder()
                 .decoder(accountDecoder)
                 .requestInterceptor(new ForwardedForInterceptor())
                 .target(Bank.class, "https://api.examplebank.com");
```

Otro ejemplo común de in interceptor podría ser autenticación, tal como usar el ya construido `BasicAuthRequestInterceptor`.

```java
Bank bank = Feign.builder()
                 .decoder(accountDecoder)
                 .requestInterceptor(new BasicAuthRequestInterceptor(username, password))
                 .target(Bank.class, "https://api.examplebank.com");
```

#### Custom @Param Expansion
Los parámetros anotados con `Param` se extienden basandose en el uso de su `toString`. Al especificar un `Param.Expander` personalizado, 
los usuarios pueden controlar este comportamiento, por ejemplo formateando fechas.

```java
@RequestLine("GET /?since={date}") Result list(@Param(value = "date", expander = DateToMillis.class) Date date);
```

#### Dynamic Query Parameters
Un parámetro Map puede ser anotado con `QueryMap` para contruir un query que use el contenido del mapa como query parameters.

```java
@RequestLine("GET /find")
V find(@QueryMap Map<String, Object> queryMap);
```

#### Static and Default Methods
Las interfaces a las cuales deseas apuntar usando Feign pueden tener metodos por defecto o estáticos (si usas Java 8+).
Estas permiten que los clientes de Feign contengan lógica que no es expresamente definida por el API en cuestión.
Por ejemplo, los metodos estáticos facilitan especificar configuraciones comunes de construcciónn del cliente; los metodos por defecto pueden ser usados para formar queries o definir parametros por defecto.

```java
interface GitHub {
  @RequestLine("GET /repos/{owner}/{repo}/contributors")
  List<Contributor> contributors(@Param("owner") String owner, @Param("repo") String repo);

  @RequestLine("GET /users/{username}/repos?sort={sort}")
  List<Repo> repos(@Param("username") String owner, @Param("sort") String sort);

  default List<Repo> repos(String owner) {
    return repos(owner, "full_name");
  }

  /**
   * Lists all contributors for all repos owned by a user.
   */
  default List<Contributor> contributors(String user) {
    MergingContributorList contributors = new MergingContributorList();
    for(Repo repo : this.repos(owner)) {
      contributors.addAll(this.contributors(user, repo.getName()));
    }
    return contributors.mergeResult();
  }

  static GitHub connect() {
    return Feign.builder()
                .decoder(new GsonDecoder())
                .target(GitHub.class, "https://api.github.com");
  }
}
```
