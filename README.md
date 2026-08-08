# GA – Trazado del flujo completo de una petición HTTP en Spring Boot 3

## Aplicaciones Web

Práctica realizada utilizando el proyecto PFC **BIOPET**, con el objetivo de observar mediante el depurador de IntelliJ IDEA el recorrido completo de una petición HTTP desde el cliente hasta PostgreSQL y su posterior respuesta en formato JSON.

---

## Objetivo

Identificar las clases y métodos reales que intervienen en el flujo de una petición HTTP dentro de una aplicación desarrollada con Spring Boot 3, utilizando breakpoints para observar el comportamiento de las capas de seguridad, controlador, servicio y acceso a datos.

---

## Petición utilizada

Para la práctica se utilizó el siguiente endpoint:

```http
GET http://localhost:8080/api/mascotas?page=0&size=13&sort=id,desc
```

La autenticación se realizó previamente mediante:

```http
POST http://localhost:8080/api/auth/login
```

El proyecto utiliza autenticación JWT mediante una cookie `HttpOnly` llamada `access_token`.

---

## Flujo observado

```text
Postman / Angular
        ↓
Tomcat embebido
        ↓
DispatcherServlet
        ↓
JwtAuthenticationFilter
        ↓
HandlerMapping
        ↓
MascotaController
        ↓
MascotaService
        ↓
MascotaRepository
        ↓
Hibernate / JPA
        ↓
PostgreSQL
        ↓
Jackson
        ↓
Respuesta JSON
```

---

## Tabla de trazado

| # | Componente | Clase / archivo | Método | Paquete |
|---|---|---|---|---|
| 1 | Cliente Angular | `MascotaApiService` | `listar(page, size, sort)` | `frontend/src/app/features/` |
| 2 | Tomcat embebido | Automático | Gestionado por Spring Boot | Spring interno |
| 3 | DispatcherServlet | `DispatcherServlet` | `doDispatch()` | `org.springframework.web.servlet` |
| 4 | Filtro JWT | `JwtAuthenticationFilter.java` | `doFilterInternal()` | `com.biopet.security` |
| 5 | HandlerMapping | `RequestMappingHandlerMapping` | `getHandler()` | Spring interno |
| 6 | Controlador | `MascotaController.java` | `listar(Pageable, UserDetails)` | `com.biopet.controller` |
| 7 | Servicio | `MascotaService.java` | `listar(Pageable, String)` | `com.biopet.service` |
| 8 | Repositorio | `MascotaRepository.java` | `findAllByActivoTrue(Pageable)` | `com.biopet.repository` |
| 9 | Serialización JSON | `MappingJackson2HttpMessageConverter` | `write()` | Spring interno |

---

## Desarrollo de la práctica

Se inició el backend de BIOPET mediante IntelliJ IDEA en modo Debug utilizando Java 21.

Posteriormente, se colocaron breakpoints en tres puntos principales del flujo:

- `JwtAuthenticationFilter.doFilterInternal()`
- `MascotaController.listar()`
- `MascotaService.listar()`

Antes de ejecutar la petición protegida, se realizó el inicio de sesión desde Postman utilizando el endpoint:

```http
POST http://localhost:8080/api/auth/login
```

El backend respondió correctamente con código HTTP `200 OK` y generó las cookies de sesión correspondientes:

- `access_token`
- `refresh_token`

Después de realizar la autenticación se ejecutó la petición:

```http
GET http://localhost:8080/api/mascotas?page=0&size=13&sort=id,desc
```

La ejecución fue detenida mediante el depurador de IntelliJ IDEA para observar el recorrido de la petición.

---

## Paso 1 – Filtro JWT

El primer breakpoint se activó dentro de:

```java
JwtAuthenticationFilter.doFilterInternal()
```

En este punto se observó la ejecución de la siguiente instrucción:

```java
Optional<String> resolvedToken = resolveToken(request);
```

Después de ejecutar esta línea mediante `Step Over (F8)`, se comprobó que la variable `resolvedToken` contenía un valor.

Esto confirmó que el filtro `JwtAuthenticationFilter` recibió correctamente el JWT enviado mediante la cookie `access_token`.

En el panel **Frames** del depurador también se pudo observar la pila de llamadas realizada por Spring y Tomcat antes de llegar al filtro de seguridad.

---

## Paso 2 – Controlador

Después de continuar la ejecución mediante `Resume (F9)`, el segundo breakpoint se activó en:

```java
MascotaController.listar()
```

La línea observada fue:

```java
return mascotaService.listar(pageable, userDetails.getUsername());
```

En el panel de variables se comprobó que Spring ya había convertido los parámetros enviados en la URL a un objeto `Pageable`.

Los valores observados fueron:

```text
pageNumber = 0
pageSize = 13
sort = id: DESC
```

También se comprobó que el usuario autenticado era:

```text
admin@biopet.ec
```

Esto demuestra que antes de ejecutar el controlador, Spring Security ya había procesado correctamente la autenticación del usuario.

---

## Paso 3 – Servicio

Después de continuar mediante `F9`, la ejecución llegó a:

```java
MascotaService.listar()
```

En este punto se observó la siguiente instrucción:

```java
Usuario usuario = usuarioRepository
        .findByEmailAndActivoTrue(email)
        .orElseThrow(() -> new RecursoNoEncontradoException("Usuario no encontrado"));
```

Dentro del panel de variables se comprobó nuevamente:

```text
email = admin@biopet.ec
pageNumber = 0
pageSize = 13
sort = id: DESC
```

Luego se evaluó el rol del usuario mediante:

```java
if (usuario.getRol() == Rol.ROLE_DUENO)
```

Como la sesión utilizada correspondía al usuario administrador, la ejecución continuó hacia:

```java
mascotaRepository.findAllByActivoTrue(pageable)
```

---

## Paso 4 – Repositorio, Hibernate y PostgreSQL

Al ejecutar la llamada:

```java
mascotaRepository.findAllByActivoTrue(pageable)
```

Spring Data JPA utilizó Hibernate para generar automáticamente la consulta SQL correspondiente.

La consulta observada en la consola fue:

```sql
select
    m1_0.id,
    m1_0.activo,
    m1_0.actualizado_en,
    m1_0.creado_en,
    m1_0.duenio_id,
    m1_0.especie,
    m1_0.fecha_nacimiento,
    m1_0.nombre,
    m1_0.raza
from
    public.mascotas m1_0
where
    m1_0.activo
order by
    m1_0.id desc
fetch
    first ? rows only
```

Hibernate también mostró el valor utilizado para el parámetro de paginación:

```text
binding parameter (1:INTEGER) <- [13]
```

Esto demuestra que el parámetro:

```text
size=13
```

enviado desde Postman fue utilizado por Hibernate como límite de registros en la consulta SQL.

---

## Paso 5 – Respuesta JSON

Después de finalizar la ejecución, Postman recibió correctamente la respuesta del backend con:

```text
200 OK
```

La respuesta fue serializada en formato JSON.

Durante la prueba se obtuvo una estructura paginada similar a:

```json
{
  "content": [],
  "pageable": {
    "pageNumber": 0,
    "pageSize": 13
  },
  "totalPages": 0,
  "totalElements": 0,
  "size": 13,
  "number": 0,
  "first": true,
  "empty": true
}
```

El arreglo `content` se encontraba vacío porque en ese momento no existían mascotas activas disponibles en la base de datos utilizada para la prueba.

Sin embargo, la petición recorrió correctamente todas las capas y terminó con código HTTP `200 OK`.

---

## Evidencias

### Evidencia 1 – Filtro JWT

Se observa la ejecución detenida dentro de `JwtAuthenticationFilter.doFilterInternal()` y la variable `resolvedToken` después de ejecutar `resolveToken(request)`.

![Filtro JWT](evidencias/01-jwt-filter-debug.png)

---

### Evidencia 2 – Controlador

Se observa la ejecución dentro de `MascotaController.listar()`.

También se visualizan los parámetros del objeto `Pageable` y el usuario autenticado.

![Controlador](evidencias/02-mascota-controller-debug.png)

---

### Evidencia 3 – Servicio

Se observa la ejecución dentro de `MascotaService.listar()`, donde se procesa el usuario autenticado y posteriormente se realiza la llamada al repositorio.

![Servicio](evidencias/03-mascota-service-debug.png)

---

### Evidencia 4 – Consulta SQL

Se observa el SQL generado automáticamente por Hibernate para consultar la tabla `public.mascotas` de PostgreSQL.

![Consulta SQL](evidencias/04-hibernate-postgresql-sql.png)

---

### Evidencia 5 – Respuesta JSON

Se observa la respuesta recibida en Postman mediante el endpoint de mascotas, con código HTTP `200 OK` y contenido serializado en JSON.

![Respuesta JSON](evidencias/05-postman-json-response.png)

---

## Preguntas de análisis

### 1. ¿Qué representa la pila de llamadas mostrada en Frames? ¿En qué orden se ejecutaron esas clases?

La pila de llamadas representa la secuencia de métodos y clases que participaron en el procesamiento de la petición hasta llegar al punto donde el depurador se encuentra detenido.

En ella aparecen componentes internos de Tomcat, Spring Security y Spring MVC junto con las clases propias del proyecto.

De forma general, el flujo observado fue:

```text
Tomcat
↓
Filtros de Spring Security
↓
JwtAuthenticationFilter
↓
DispatcherServlet
↓
HandlerMapping
↓
MascotaController
```

El panel `Frames` permite observar cómo el framework ejecuta automáticamente diferentes componentes antes de llegar al código desarrollado por el equipo.

---

### 2. ¿Qué está haciendo Spring cuando aparecen clases como TransactionInterceptor y CglibAopProxy?

Spring utiliza proxies e interceptores para aplicar comportamientos adicionales antes de ejecutar directamente los métodos del servicio.

En este caso, `MascotaService.listar()` utiliza:

```java
@Transactional(readOnly = true)
```

Por lo tanto, Spring puede utilizar componentes como:

```text
TransactionInterceptor
CglibAopProxy
```

para preparar y administrar la transacción antes de ejecutar la lógica del servicio.

Esto es importante porque permite controlar automáticamente el inicio, confirmación o cancelación de las operaciones relacionadas con la base de datos.

---

### 3. ¿Por qué aparece SimpleJpaRepository si esa clase nunca fue creada por el equipo?

`MascotaRepository` es una interfaz que utiliza Spring Data JPA.

El equipo únicamente define algo similar a:

```java
public interface MascotaRepository extends JpaRepository<Mascota, Long>
```

Spring genera automáticamente una implementación en tiempo de ejecución.

La implementación base utilizada por Spring Data JPA es:

```text
SimpleJpaRepository
```

Por esta razón, el depurador puede mostrar esta clase aunque no exista como archivo Java creado directamente dentro del proyecto BIOPET.

---

### 4. ¿Qué sucede si la petición se realiza sin un JWT válido?

Las peticiones protegidas pasan primero por la cadena de seguridad de Spring.

En BIOPET, una de las clases principales encargadas del procesamiento del token es:

```java
JwtAuthenticationFilter
```

El filtro intenta obtener y validar el JWT antes de que la petición llegue al controlador.

Si la solicitud no contiene una autenticación válida y el recurso necesita autenticación, Spring Security puede detener el flujo antes de ejecutar el método del controlador y devolver una respuesta HTTP de no autorizado.

---

### 5. ¿Hibernate obtiene todos los campos de la entidad o solamente los que necesita el DTO?

Durante la práctica se observó que Hibernate generó una consulta con los campos mapeados de la entidad `Mascota`:

```sql
m1_0.id,
m1_0.activo,
m1_0.actualizado_en,
m1_0.creado_en,
m1_0.duenio_id,
m1_0.especie,
m1_0.fecha_nacimiento,
m1_0.nombre,
m1_0.raza
```

Posteriormente, el servicio transforma las entidades obtenidas a objetos `MascotaResponse`.

Por lo tanto, en esta consulta Hibernate obtiene los campos mapeados de la entidad y después se realiza la transformación al DTO.

En una tabla con una cantidad muy grande de columnas, esto podría generar un costo adicional porque la base de datos estaría enviando información que posiblemente no será utilizada finalmente por el DTO.

---

## Resultado obtenido

La práctica permitió observar de forma directa el recorrido completo de una petición HTTP dentro del proyecto BIOPET.

Mediante los breakpoints del depurador de IntelliJ IDEA se comprobó el funcionamiento del filtro JWT, el controlador y el servicio.

También se observó cómo Spring Data JPA utiliza Hibernate para generar automáticamente la consulta SQL enviada a PostgreSQL.

La petición utilizada permitió comprobar los parámetros de paginación enviados desde Postman, el usuario autenticado y la consulta ejecutada sobre la tabla `mascotas`.

Finalmente, el backend respondió correctamente con código HTTP `200 OK` y Spring convirtió el resultado en una respuesta JSON.

---

## Conclusión

El trazado realizado permitió comprender de manera práctica cómo Spring Boot procesa una petición HTTP internamente.

Aunque desde el cliente solamente se realiza una solicitud a un endpoint, dentro del backend intervienen diferentes componentes encargados de la seguridad, enrutamiento, lógica de negocio, acceso a datos y serialización de la respuesta.

El uso del depurador permitió comprobar que la petición pasó por `JwtAuthenticationFilter`, `MascotaController`, `MascotaService` y `MascotaRepository` antes de ejecutar la consulta en PostgreSQL.

De esta forma se pudo relacionar el funcionamiento del patrón MVC con la arquitectura real utilizada en el PFC BIOPET.

---

## Flujo final identificado

```text
Postman / Angular
        ↓
Tomcat
        ↓
DispatcherServlet
        ↓
Spring Security
        ↓
JwtAuthenticationFilter
        ↓
HandlerMapping
        ↓
MascotaController.listar()
        ↓
MascotaService.listar()
        ↓
MascotaRepository.findAllByActivoTrue()
        ↓
Hibernate
        ↓
PostgreSQL
        ↓
MascotaResponse
        ↓
Jackson
        ↓
JSON
        ↓
Postman – 200 OK
```
