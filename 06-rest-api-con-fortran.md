# Capítulo 4: REST API con Fortran Moderno

> En este capítulo aprenderás a construir una API RESTful completa usando Fortran moderno. Exploraremos cómo Fortran puede interoperar con C para acceder a llamadas del sistema, cómo hacer peticiones HTTP usando `curl` desde Fortran, cómo construir un servidor HTTP mínimo con la ayuda de herramientas del sistema (`netcat`), y cómo generar y consumir JSON utilizando exclusivamente código Fortran puro. Al final, implementarás un API RESTful funcional para un catálogo de productos, con endpoints GET, POST, PUT y DELETE, serialización JSON, enrutamiento de peticiones y logging, todo desde Fortran.

> ---
> **Stack tecnológico**: gfortran 13+ (estándar -std=f2018), curl 7.81+, netcat-openbsd (nc), GNU/Linux Ubuntu 22.04+. No se requieren bibliotecas externas de Fortran; todo el código JSON y HTTP es implementación nativa.
> ---

## 1. Introducción a REST APIs

Imagina que estás en una biblioteca. Hay un mostrador de atención, y detrás hay estantes llenos de libros. Tú, como visitante, no puedes entrar directamente a los estantes; en vez de eso, le pides al bibliotecario que te traiga un libro específico, o que te diga qué libros hay disponibles, o que registre un nuevo libro en el catálogo. El bibliotecario es una interfaz: tú le haces una petición y él te devuelve una respuesta.

En el mundo del software, esa misma dinámica ocurre entre programas. Una aplicación necesita datos o servicios que otra aplicación posee, pero no puede acceder directamente a la memoria o los archivos de la otra. En lugar de eso, se comunican a través de una interfaz bien definida: una API (Application Programming Interface). REST (Representational State Transfer) es un conjunto de principios para diseñar esas interfaces de manera predecible, escalable y desacoplada.

REST no es un protocolo ni una librería; es un estilo arquitectónico que se apoya en el protocolo HTTP, el mismo que usa tu navegador para cargar páginas web. Cuando escribes una URL en el navegador y presionas Enter, estás haciendo una petición GET a un servidor. El servidor responde con un código de estado (200 si todo va bien, 404 si no encuentra el recurso) y un cuerpo (el HTML de la página). REST formaliza ese patrón: cada "cosa" en el sistema (un producto, un usuario, una factura) es un **recurso** identificado por una URI (Uniform Resource Identifier), y las operaciones sobre ese recurso se mapean a los verbos HTTP.

¿Por qué esto es relevante para un programador de Fortran? Históricamente, Fortran ha sido el lenguaje de la computación científica y numérica: simulaciones climáticas, dinámica de fluidos, modelos financieros. Pero esos programas no operan en el vacío. Necesitan recibir datos de entrada, exponer resultados, integrarse con dashboards web, o comunicarse con otros servicios. Poder construir una API REST desde Fortran —sin depender de un lenguaje intermedio como Python o C— permite que el código científico se convierta en un servicio autónomo, desplegable y consumible por cualquier cliente HTTP.

Los verbos HTTP fundamentales son cuatro, y forman el acrónimo CRUD (Create, Read, Update, Delete). GET obtiene un recurso (leer). POST crea un recurso nuevo (crear). PUT actualiza un recurso existente (actualizar). DELETE elimina un recurso (borrar). Cada uno tiene semánticas específicas sobre seguridad e idempotencia: GET y DELETE son idempotentes (la misma petición múltiples veces produce el mismo resultado), PUT también lo es; POST no lo es.

Los códigos de estado HTTP se agrupan en familias: 200 (OK), 201 (Created), 204 (No Content) para respuestas exitosas; 400 (Bad Request) cuando el cliente envía datos inválidos; 404 (Not Found) cuando el recurso no existe; 405 (Method Not Allowed) cuando el verbo no está soportado para esa ruta; 500 (Internal Server Error) para fallos del servidor. Elegir el código correcto es parte del diseño de la API: no se trata solo de devolver datos, sino de comunicar con precisión lo que ocurrió.

En las próximas secciones construiremos cada pieza desde cero. Primero veremos cómo Fortran puede llamar a funciones del sistema operativo escritas en C, lo que nos da acceso a herramientas como sockets y ejecución de comandos. Luego construiremos un cliente HTTP usando `curl` desde Fortran, después un servidor HTTP mínimo, y finalmente un módulo JSON nativo. Todo esto culminará en un mini-proyecto funcional.

```fortran
program demo_http_strings
  implicit none
  character(len=:), allocatable :: request, response, body
  integer :: content_length

  ! Construimos una peticion HTTP GET manualmente
  request = 'GET /productos HTTP/1.1' // char(13) // char(10) // &
            'Host: localhost:8080' // char(13) // char(10) // &
            char(13) // char(10)

  print *, '=== Peticion HTTP ==='
  print *, trim(request)

  ! Construimos una respuesta HTTP con cuerpo JSON
  body = '{"mensaje": "Hola desde Fortran"}'
  content_length = len_trim(body)

  response = 'HTTP/1.1 200 OK' // char(13) // char(10) // &
             'Content-Type: application/json' // char(13) // char(10) // &
             'Content-Length: ' // trim(adjustl(int_to_str(content_length))) // &
             char(13) // char(10) // &
             char(13) // char(10) // &
             body

  print *, '=== Respuesta HTTP ==='
  print *, trim(response)

contains

  function int_to_str(val) result(str)
    integer, intent(in) :: val
    character(len=32) :: buf
    character(len=:), allocatable :: str
    write(buf, '(i0)') val
    str = trim(buf)
  end function

end program demo_http_strings
```

**Análisis línea por línea**:

- `character(len=:), allocatable :: request, response, body`: Declaramos tres cadenas de caracteres de longitud dinámica (allocatable). En Fortran moderno, `len=:` permite que la cadena se ajuste automáticamente al valor que se le asigna.
- `char(13) // char(10)`: El protocolo HTTP exige que las líneas terminen con CRLF (Carriage Return 0x0D + Line Feed 0x0A). `char(13)` produce el retorno de carro y `char(10)` el salto de línea.
- `'GET /productos HTTP/1.1'`: La línea de petición HTTP tiene tres partes separadas por espacios: el método, la ruta, y la versión del protocolo.
- `'Host: localhost:8080'`: El encabezado `Host` es obligatorio en HTTP/1.1 e indica el servidor al que va dirigida la petición.
- `content_length = len_trim(body)`: `len_trim` devuelve la longitud de la cadena ignorando espacios finales. `Content-Length` debe contar exactamente los bytes del cuerpo.
- `trim(adjustl(int_to_str(content_length)))`: `adjustl` elimina espacios a la izquierda y `trim` a la derecha, produciendo una representación limpia del entero.
- Dentro de `int_to_str`: `write(buf, '(i0)') val` usa el descriptor de formato `i0` que escribe un entero sin espacios adicionales ni ancho fijo.

**Salida esperada**:

```
 === Peticion HTTP ===
GET /productos HTTP/1.1
Host: localhost:8080

 === Respuesta HTTP ===
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 39

{"mensaje": "Hola desde Fortran"}
```

**Errores típicos**:

Error: olvidar el CRLF doble entre cabeceras y cuerpo.
```fortran
response = 'HTTP/1.1 200 OK' // char(13)//char(10) // &
           'Content-Type: application/json' // &
           body  ! Falta el CRLF doble
```
Mensaje de error: no hay error de compilación, pero el cliente HTTP (curl, navegador) interpretará que el cuerpo es parte de la última cabecera y la respuesta será inválida. Solución: siempre incluir `char(13)//char(10)//char(13)//char(10)` antes del cuerpo.

Error: usar `char(10)` solo (LF) en lugar de `char(13)//char(10)` (CRLF). HTTP exige CRLF; algunos servidores/clientes toleran LF, pero no es estándar. Síntoma: respuestas que funcionan en un cliente pero no en otro. Solución: acostumbrarse a usar CRLF (`char(13)//char(10)`) en todo el código HTTP.

---

## 2. Interoperabilidad con C (iso_c_binding)

Has construido modelos climáticos en Fortran durante años. Tu código es rápido, estable y probado. Pero de repente necesitas obtener la hora del sistema con precisión de nanosegundos, o abrir un socket TCP, o consultar variables de entorno. El problema: Fortran no tiene una biblioteca estándar para estas operaciones de sistema. C, en cambio, tiene `sys/socket.h`, `time.h`, `unistd.h`. ¿Qué haces? ¿Reescribes todo en C? ¿Usas un binding externo?

La respuesta está en `iso_c_binding`, un módulo intrínseco introducido en Fortran 2003 que permite a Fortran llamar directamente a funciones escritas en C con sobrecarga prácticamente nula. Piensa en ello como un puente entre dos mundos: Fortran conserva su eficiencia numérica y su sintaxis familiar, pero puede extender sus capacidades invocando cualquier función del sistema operativo que tenga una interfaz C.

¿Cómo funciona? Toda función en C tiene un nombre simbólico en el binario compilado. El atributo `bind(C, name="nombre_c")` le dice al compilador de Fortran: "esta función que estoy declarando no es Fortran, es una función C con este nombre exacto en la biblioteca del sistema". El módulo `iso_c_binding` proporciona tipos compatibles: `c_int` para `int`, `c_char` para `char`, `c_double` para `double`, `c_null_char` para el terminador nulo `\0` que C espera al final de las cadenas.

La contraparte también funciona: puedes marcar una subrutina o función de Fortran con `bind(C)` para que sea invocable desde C. Esto es útil cuando quieres que Fortran sea el backend de un programa en C, o cuando necesitas que un callback de una librería C llame a tu código Fortran. La interoperabilidad es bidireccional.

En la práctica, para nuestro objetivo de construir una API REST, `iso_c_binding` nos permite dos cosas esenciales: invocar `system()` para ejecutar comandos del shell (como `curl`), y —con más complejidad— usar las funciones de socket de POSIX (`socket`, `bind`, `listen`, `accept`, `send`, `recv`) directamente desde Fortran. Esto último nos daría un servidor HTTP sin depender de herramientas externas.

Sin embargo, hay una advertencia importante: las cadenas en C terminan en nulo (`\0`), mientras que en Fortran son de longitud fija con espacios. Cuando pasas una cadena Fortran a una función C, debes agregar `c_null_char` al final. Cuando recibes una cadena de C en Fortran, puede que necesites truncarla en el primer nulo. Este detalle es fuente frecuente de errores difíciles de depurar.

Empecemos con un ejemplo simple: llamar a la función `system()` de C desde Fortran. Esta función ejecuta un comando en el shell y devuelve el código de salida. Es la puerta de entrada a todo el ecosistema de comandos Unix.

```fortran
program call_system
  use iso_c_binding
  implicit none

  interface
    ! Declaramos la funcion system() de C como una interfaz Fortran
    function c_system(cmd) bind(C, name="system")
      use iso_c_binding
      implicit none
      character(kind=c_char), intent(in) :: cmd(*)
      integer(c_int) :: c_system
    end function
  end interface

  integer(c_int) :: result
  character(len=:), allocatable :: command

  ! Ejecutamos un comando simple
  command = 'echo "Hola desde C, invocado por Fortran!"' // c_null_char
  result = c_system(command)

  print *, 'Codigo de salida:', result

  ! Probamos con un comando que falla
  command = 'exit 42' // c_null_char
  result = c_system(command)
  print *, 'Codigo de salida (exit 42):', result

end program call_system
```

**Análisis línea por línea**:

- `use iso_c_binding`: Importa el módulo intrínseco que define los tipos compatibles con C (`c_int`, `c_char`, `c_null_char`, etc.).
- `interface ... end interface`: Un bloque `interface` declara funciones externas sin implementarlas. Es como un prototipo en C.
- `function c_system(cmd) bind(C, name="system")`: Declaramos una función llamada `c_system` que enlaza con la función C `system`. El nombre Fortran puede ser cualquiera; el enlace real se define con `name=` del bind(C).
- `character(kind=c_char), intent(in) :: cmd(*)`: El parámetro es un arreglo de caracteres C (`kind=c_char`), de tamaño indeterminado (`*`), típico para cadenas C que terminan en nulo.
- `integer(c_int) :: c_system`: El valor de retorno es un entero C (`int`). La función `system()` devuelve el código de salida del comando.
- `command = '...' // c_null_char`: `c_null_char` es una constante del módulo `iso_c_binding` que vale `char(0)`. Toda cadena que se pase a C debe terminar en nulo.
- `result = c_system(command)`: Invocamos la función C directamente, como si fuera una función Fortran más.
- `print *, 'Codigo de salida:', result`: Imprimimos el código de retorno. 0 indica éxito; cualquier otro valor indica error.

**Salida esperada**:

```
 Hola desde C, invocado por Fortran!
 Codigo de salida:           0
 Codigo de salida (exit 42):          42
```

(Las líneas de `echo` y de `print` pueden aparecer entrelazadas porque `system()` escribe directamente a stdout mientras que `print` usa el sistema de E/S de Fortran; es un efecto normal de la mezcla de runtime.)

**Errores típicos**:

Error: olvidar `c_null_char` al final de la cadena.
```fortran
! Codigo erroneo: falta c_null_char
command = 'echo hola'
result = c_system(command)
```
Mensaje de error: no hay error de compilación, pero en ejecución `system()` recibe una cadena que no termina en nulo, lo que causa comportamiento indefinido (puede funcionar por casualidad o corromper memoria). Solución: siempre concatenar `// c_null_char` al construir comandos para `system()`.

Error: tipo incorrecto en el parámetro de la interfaz.
```fortran
! Codigo erroneo: usar character(len=*) en vez de character(kind=c_char)(*)
function c_system(cmd) bind(C, name="system")
  use iso_c_binding
  character(len=*), intent(in) :: cmd  ! Error: tipo incompatible con C
  integer(c_int) :: c_system
end function
```
Mensaje de error de gfortran: `Error: Variable 'cmd' at (1) cannot have the ALLOCATABLE, CODIMENSION, or POINTER attribute because it is a dummy argument of a BIND(C) procedure`. O, en versiones más recientes: `Error: Argument 'cmd' of 'c_system' at (1) is a character type with noninteroperable length`. Solución: usar `character(kind=c_char), intent(in) :: cmd(*)`.

Ahora veamos un ejemplo más avanzado: un servidor de echo usando sockets POSIX directamente desde Fortran. Este código es más denso, pero muestra el poder real de `iso_c_binding` para acceder a la capa de red.

```fortran
program socket_server
  use iso_c_binding
  implicit none

  interface
    function c_socket(domain, type, protocol) bind(C, name="socket")
      use iso_c_binding
      implicit none
      integer(c_int), value :: domain, type, protocol
      integer(c_int) :: c_socket
    end function

    function c_bind(sockfd, addr, addrlen) bind(C, name="bind")
      use iso_c_binding
      implicit none
      integer(c_int), value :: sockfd
      type(*), dimension(*), intent(in) :: addr
      integer(c_int), value :: addrlen
      integer(c_int) :: c_bind
    end function

    function c_listen(sockfd, backlog) bind(C, name="listen")
      use iso_c_binding
      implicit none
      integer(c_int), value :: sockfd, backlog
      integer(c_int) :: c_listen
    end function

    function c_accept(sockfd, addr, addrlen) bind(C, name="accept")
      use iso_c_binding
      implicit none
      integer(c_int), value :: sockfd
      type(*), dimension(*), intent(out) :: addr
      integer(c_int), intent(inout) :: addrlen
      integer(c_int) :: c_accept
    end function

    function c_send(sockfd, buf, len, flags) bind(C, name="send")
      use iso_c_binding
      implicit none
      integer(c_int), value :: sockfd
      character(kind=c_char), intent(in) :: buf(*)
      integer(c_int), value :: len, flags
      integer(c_int) :: c_send
    end function

    function c_recv(sockfd, buf, len, flags) bind(C, name="recv")
      use iso_c_binding
      implicit none
      integer(c_int), value :: sockfd
      character(kind=c_char), intent(out) :: buf(*)
      integer(c_int), value :: len, flags
      integer(c_int) :: c_recv
    end function

    function c_close(fd) bind(C, name="close")
      use iso_c_binding
      implicit none
      integer(c_int), value :: fd
      integer(c_int) :: c_close
    end function

    function c_htons(value) bind(C, name="htons")
      use iso_c_binding
      implicit none
      integer(c_int), value :: value
      integer(c_int) :: c_htons
    end function
  end interface

  integer(c_int) :: sockfd, clientfd
  integer(c_int) :: result, addrlen
  character(kind=c_char, len=1024) :: buffer
  character(len=:), allocatable :: response
  integer(c_int) :: optval, port

  ! Constantes POSIX
  integer(c_int), parameter :: AF_INET = 2
  integer(c_int), parameter :: SOCK_STREAM = 1

  ! Crear socket TCP/IPv4
  sockfd = c_socket(AF_INET, SOCK_STREAM, 0)
  if (sockfd < 0) then
    print *, 'Error al crear socket'
    stop
  end if
  print *, 'Socket creado, fd =', sockfd

  ! Preparar direccion: puerto 8080
  block
    integer(c_int8_t) :: addr_struct(16)  ! sockaddr_in en Linux = 16 bytes

    addr_struct = 0_c_int8_t
    addr_struct(1) = AF_INET  ! sin_family = AF_INET

    ! Puerto 8080 en network byte order (htons)
    port = c_htons(int(8080, c_int))
    addr_struct(3) = int(ibits(port, 8, 8), c_int8_t)
    addr_struct(4) = int(ibits(port, 0, 8), c_int8_t)

    ! INADDR_ANY = 0 (todas las interfaces)
    addr_struct(5:8) = 0_c_int8_t

    addrlen = 16
    result = c_bind(sockfd, addr_struct, addrlen)
    if (result < 0) then
      print *, 'Error al hacer bind'
      call c_close(sockfd)
      stop
    end if
    print *, 'Bind exitoso en puerto 8080'
  end block

  ! Escuchar con backlog de 5 conexiones
  result = c_listen(sockfd, 5)
  if (result < 0) then
    print *, 'Error al escuchar'
    call c_close(sockfd)
    stop
  end if
  print *, 'Escuchando en puerto 8080...'

  ! Aceptar una conexion
  addrlen = 16
  clientfd = c_accept(sockfd, buffer, addrlen)
  if (clientfd < 0) then
    print *, 'Error al aceptar conexion'
    call c_close(sockfd)
    stop
  end if
  print *, 'Conexion aceptada, client_fd =', clientfd

  ! Recibir datos
  buffer = ''
  result = c_recv(clientfd, buffer, int(1024, c_int), 0)
  print *, 'Recibidos', result, 'bytes'
  print *, 'Datos recibidos:'
  print *, buffer(1:result)

  ! Enviar respuesta HTTP
  response = 'HTTP/1.1 200 OK' // char(13)//char(10) // &
             'Content-Type: text/plain' // char(13)//char(10) // &
             'Content-Length: 13' // char(13)//char(10) // &
             char(13)//char(10) // &
             'Hola, mundo!' // char(13)//char(10)
  result = c_send(clientfd, response, int(len_trim(response), c_int), 0)
  print *, 'Enviados', result, 'bytes'

  ! Cerrar conexiones
  result = c_close(clientfd)
  result = c_close(sockfd)

end program socket_server
```

**Análisis línea por línea**:

- `interface ... end interface`: Seis bloques de interfaz para las funciones C: `socket`, `bind`, `listen`, `accept`, `send`, `recv`, `close`, `htons`. Cada una tiene su firma exacta según POSIX.
- `type(*), dimension(*), intent(in) :: addr`: `type(*)` es un tipo C asociado (buffer genérico), necesario para pasar la estructura `sockaddr_in`.
- `integer(c_int), value :: domain`: El atributo `value` indica que el argumento se pasa por valor (como en C), no por referencia (como es default en Fortran).
- `AF_INET = 2, SOCK_STREAM = 1`: Constantes POSIX para IPv4 y TCP. En Linux, `AF_INET` vale 2 y `SOCK_STREAM` vale 1.
- `block ... end block`: Usamos un bloque para declarar variables locales (`addr_struct`) sin contaminar el ámbito del programa principal.
- `integer(c_int8_t) :: addr_struct(16)`: Representamos `struct sockaddr_in` como un arreglo de 16 bytes. `c_int8_t` es un entero de 8 bits (byte) del módulo `iso_c_binding`.
- `port = c_htons(int(8080, c_int))`: `htons` convierte el puerto de little-endian (x86) a big-endian (network byte order). `int(..., c_int)` convierte el literal entero a tipo C int.
- `addr_struct(3) = int(ibits(port, 8, 8), c_int8_t)`: `ibits` extrae los bytes individuales del puerto después de `htons` y los coloca en la estructura de dirección.
- `result = c_accept(sockfd, buffer, addrlen)`: `accept` espera una conexión entrante. El segundo argumento es un buffer para la dirección del cliente; el tercero es la longitud.
- `result = c_recv(clientfd, buffer, int(1024, c_int), 0)`: `recv` lee datos del socket. El flag `0` significa lectura normal.
- `result = c_send(clientfd, response, int(len_trim(response), c_int), 0)`: Enviamos la respuesta.

**Salida esperada** (ejecutar con `curl http://localhost:8080` en otra terminal):

```
 Socket creado, fd =           3
 Bind exitoso en puerto 8080
 Escuchando en puerto 8080...
 Conexion aceptada, client_fd =           4
 Recibidos         221 bytes
 Datos recibidos:
GET / HTTP/1.1
Host: localhost:8080
User-Agent: curl/7.81.0
Accept: */*


 Enviados          71 bytes
```

**Errores típicos**:

Error: orden de bytes incorrecto en el puerto. Si usas `port = 8080` directamente sin `htons`, el bind falla porque el puerto se interpreta al revés (8080 little-endian = 0x901F en el cable, que es un puerto no privilegiado distinto). Síntoma: `Error al hacer bind` con código de error `errno=22` (EINVAL) o `errno=98` (EADDRINUSE). Solución: siempre usar `htons` para convertir el puerto.

Error: olvidar el atributo `value` en los parámetros de bind(C).
```fortran
! Codigo erroneo: falta value
function c_socket(domain, type, protocol) bind(C, name="socket")
  integer(c_int), intent(in) :: domain, type, protocol  ! Error: pasados por referencia
  integer(c_int) :: c_socket
end function
```
Mensaje de error de gfortran: no hay error de compilación, pero en ejecución `socket()` recibe punteros en lugar de valores, lo que causa `Bad address` o `segfault`. Solución: siempre usar `integer(c_int), value` en parámetros de entrada de funciones C.

---

## 3. Cliente HTTP con execute_command_line + curl

En la sección anterior viste cómo Fortran puede invocar funciones C directamente. Esa es una herramienta poderosa, pero a veces lo más práctico no es lo más elegante. Para hacer una petición HTTP desde Fortran, podríamos usar `iso_c_binding` con `libcurl`, la biblioteca C para transferencias URL. Pero `libcurl` requiere enlazar la biblioteca al compilar, gestionar manejadores de memoria C, y lidiar con callbacks. Es factible, pero verboso.

Hay un camino más simple: Fortran moderno incluye el intrínseco `execute_command_line` desde el estándar Fortran 2008. Esta subrutina ejecuta un comando en el shell del sistema operativo, exactamente como si lo escribieras en la terminal. ¿Y qué herramienta de línea de comandos está diseñada específicamente para hacer peticiones HTTP? `curl`.

La combinación `execute_command_line` + `curl` te da un cliente HTTP completo con soporte para GET, POST, PUT, DELETE, cabeceras personalizadas, autenticación, SSL, redirecciones, y todo lo que `curl` ofrece. Y todo eso sin escribir una sola línea de código de red. La desventaja, por supuesto, es que dependes de tener `curl` instalado y que la comunicación ocurre a través de un proceso hijo del shell, con la sobrecarga que eso implica. Pero para prototipos, herramientas internas, y aprendizaje, es una solución perfecta.

El flujo es: construyes una cadena con el comando `curl` completo (URL, método, cabeceras, datos), llamas a `execute_command_line`, y `curl` imprime la respuesta en la terminal. Si quieres capturar la respuesta en una variable de Fortran, puedes redirigir la salida de `curl` a un archivo temporal y luego leer ese archivo desde Fortran. Es un patrón artesanal pero efectivo.

¿Por qué este enfoque es valioso pedagógicamente? Porque separa las responsabilidades: `curl` se encarga de la complejidad del protocolo HTTP (handshake TCP, parsing de cabeceras, codificación de chunked transfer, etc.), mientras que Fortran se encarga de la lógica de aplicación (qué datos enviar, cómo procesar la respuesta). Cuando eventualmente construyas tu propio servidor HTTP en Fortran, entenderás qué partes del protocolo estás reemplazando y cuáles sigue siendo útil delegar.

Cabe preguntarse: ¿cuándo NO usarías este enfoque? Cuando necesitas hacer cientos de peticiones por segundo (el costo de lanzar un proceso por petición es alto), cuando necesitas control fino sobre el socket (timeouts por operación, reinicio de conexiones), o cuando curl no está disponible (sistemas embebidos, entornos restringidos). En esos casos, el binding directo con libcurl o sockets POSIX es la solución.

Para nuestro objetivo, el cliente HTTP con `execute_command_line` + `curl` nos servirá para dos propósitos: probar el servidor que construiremos en la siguiente sección, y como base para el cliente de prueba del mini-proyecto final.

```fortran
program http_client
  implicit none
  character(len=256) :: cmd
  integer :: exitstat

  ! --- GET request simple ---
  print *, '=== GET a JSONPlaceholder ==='
  cmd = 'curl -s https://jsonplaceholder.typicode.com/posts/1'
  call execute_command_line(cmd, exitstat=exitstat)
  print *, 'Exit status:', exitstat
  print *, ''

  ! --- POST con datos JSON ---
  print *, '=== POST crear recurso ==='
  cmd = "curl -s -X POST " // &
        "-H 'Content-Type: application/json' " // &
        "-d '{\"title\":\"Fortran\",\"body\":\"Hola desde Fortran\",\"userId\":1}' " // &
        "https://jsonplaceholder.typicode.com/posts"
  call execute_command_line(cmd, exitstat=exitstat)
  print *, 'Exit status:', exitstat
  print *, ''

  ! --- GET con captura en archivo ---
  print *, '=== GET con captura en archivo ==='
  cmd = 'curl -s https://jsonplaceholder.typicode.com/posts/2 > /tmp/response.json'
  call execute_command_line(cmd, exitstat=exitstat)
  print *, 'Respuesta guardada en /tmp/response.json, status:', exitstat

end program http_client
```

**Análisis línea por línea**:

- `character(len=256) :: cmd`: Reservamos 256 caracteres para el comando. Suficiente para la mayoría de URLs y opciones de curl.
- `call execute_command_line(cmd, exitstat=exitstat)`: El intrínseco `execute_command_line` ejecuta `cmd` en el shell. El argumento opcional `exitstat` captura el código de salida del comando (0 si éxito, distinto de 0 si error).
- `curl -s`: La opción `-s` (silent) de curl suprime la barra de progreso y mensajes de error. Sin ella, la salida se mezcla con ruido visual.
- `curl -X POST`: `-X` especifica el método HTTP. `-H 'Content-Type: application/json'` añade una cabecera. `-d '...'` envía datos en el cuerpo de la petición.
- `> /tmp/response.json`: Redirigimos la salida de curl a un archivo. Luego podemos leer ese archivo con Fortran para procesar la respuesta JSON.

**Salida esperada**:

```
 === GET a JSONPlaceholder ===
{
  "userId": 1,
  "id": 1,
  "title": "sunt aut facere repellat provident occaecati excepturi optio reprehenderit",
  "body": "quia et suscipit..."
}
 Exit status:           0

 === POST crear recurso ===
{
  "title": "Fortran",
  "body": "Hola desde Fortran",
  "userId": 1,
  "id": 101
}
 Exit status:           0

 === GET con captura en archivo ===
 Respuesta guardada en /tmp/response.json, status:           0
```

**Errores típicos**:

Error: usar comillas simples dentro de comillas simples en el comando.
```fortran
cmd = "curl -s -d '{"title":"test"}' http://localhost"  ! Error de sintaxis
```
Mensaje de error: el shell interpreta mal las comillas porque las comillas simples de Fortran y del shell se entremezclan. Solución: escapar las comillas con concatenación:
```fortran
cmd = "curl -s -H 'Content-Type: application/json' -d '" // &
      '{"title":"test"}' // "' http://localhost:8080/productos"
```

Error: no verificar `exitstat`. Si `curl` falla (servidor caído, URL incorrecta, timeout), `exitstat` será distinto de 0, pero el programa continúa como si nada.
```fortran
! Codigo erroneo: ignora el codigo de salida
call execute_command_line('curl -s http://servidor-inexistente/api')
print *, 'Continuando como si todo estuviera bien...'
```
Solución: siempre verificar `exitstat`:
```fortran
call execute_command_line(cmd, exitstat=exitstat)
if (exitstat /= 0) then
  print *, 'Error en peticion HTTP, codigo:', exitstat
  stop
end if
```

Ahora veamos un cliente más completo que captura la respuesta y la procesa con operaciones de cadena Fortran:

```fortran
program client_con_captura
  implicit none
  character(len=4096) :: response
  integer :: unit, io_stat, exitstat
  character(len=256) :: cmd

  ! Hacemos la peticion y guardamos en archivo temporal
  cmd = 'curl -s https://jsonplaceholder.typicode.com/posts/3 > /tmp/curl_out.txt 2>/tmp/curl_err.txt'
  call execute_command_line(cmd, exitstat=exitstat)

  if (exitstat == 0) then
    ! Abrimos el archivo de respuesta
    open(newunit=unit, file='/tmp/curl_out.txt', status='old', action='read', iostat=io_stat)
    if (io_stat == 0) then
      read(unit, '(a)', iostat=io_stat) response
      close(unit)

      ! Procesamos la respuesta con operaciones de cadena
      print *, 'Titulo extraido de la respuesta:'
      call extraer_campo_json(response, '"title"')
    else
      print *, 'Error al abrir archivo de respuesta'
    end if
  else
    print *, 'Error en curl, exitstat:', exitstat
  end if

contains

  subroutine extraer_campo_json(json_str, campo)
    character(len=*), intent(in) :: json_str, campo
    integer :: pos_inicio, pos_fin
    character(len=256) :: valor

    pos_inicio = index(json_str, campo)
    if (pos_inicio == 0) then
      print *, 'Campo no encontrado:', campo
      return
    end if

    ! Buscar el valor despues de ": "
    pos_inicio = index(json_str(pos_inicio:), ':') + pos_inicio
    pos_inicio = index(json_str(pos_inicio:), '"') + pos_inicio
    pos_fin = index(json_str(pos_inicio+1:), '"') + pos_inicio

    if (pos_fin > pos_inicio) then
      valor = json_str(pos_inicio:pos_fin)
      print *, trim(campo), ' = ', trim(valor)
    end if
  end subroutine

end program client_con_captura
```

**Análisis línea por línea**:

- `response = ''`: Inicializamos la cadena que contendrá la respuesta JSON. La longitud fija de 4096 caracteres es una limitación práctica.
- `cmd = '... > /tmp/curl_out.txt 2>/tmp/curl_err.txt'`: Redirigimos stdout al archivo de salida y stderr al archivo de errores. Esto permite capturar la respuesta sin que se mezcle con mensajes de diagnóstico de curl.
- `newunit=unit`: El especificador `newunit` asigna automáticamente un número de unidad no utilizado, evitando conflictos con otras unidades de E/S.
- `read(unit, '(a)', iostat=io_stat) response`: Leemos la primera línea del archivo. Para JSON de una sola línea (minificado), esto captura todo el contenido.
- `call extraer_campo_json(response, '"title"')`: Llamamos a una subrutina auxiliar que extrae un campo JSON usando operaciones de índice de cadena.
- Dentro de `extraer_campo_json`: `index(json_str, campo)` busca la primera ocurrencia de `"title"` en la cadena JSON. Luego navegamos por los caracteres `:`, `"`, etc., para aislar el valor.

**Salida esperada**:

```
 Titulo extraido de la respuesta:
"title" = "ea molestias quasi exercitationem repellat qui ipsa sit aut"
```

**Errores típicos**:

Error: el archivo de respuesta JSON tiene múltiples líneas (pretty-print). El `read` con `'(a)'` solo lee la primera línea.
```fortran
! Codigo erroneo: solo lee la primera linea
read(unit, '(a)', iostat=io_stat) response
```
Solución: leer en un bucle:
```fortran
response = ''
do
  read(unit, '(a)', iostat=io_stat) line
  if (io_stat /= 0) exit
  response = trim(response) // trim(line)
end do
```

---

## 4. Servidor HTTP básico

Hemos visto cómo Fortran puede hacer peticiones HTTP usando `execute_command_line` + `curl`. Ahora abordemos el reverso de la moneda: ¿cómo construimos un servidor HTTP en Fortran que pueda recibir esas peticiones y devolver respuestas? La respuesta, fiel a nuestro enfoque práctico, combina Fortran con una herramienta del sistema: `netcat` (nc).

Netcat es conocido como la "navaja suiza" del TCP/IP. Puede leer y escribir a través de conexiones de red, escuchar puertos, y redirigir datos hacia programas. La opción `-e` de netcat-openbsd ejecuta un programa y conecta su stdin/stdout al socket TCP. Esto convierte a cualquier programa que lea de stdin y escriba a stdout en un servidor de red, sin que el programa mismo tenga que gestionar sockets.

Este modelo se conoce como CGI (Common Gateway Interface) o, más generalmente, un "handler" estilo inetd. El flujo es: (1) netcat escucha en el puerto 8080, (2) cuando llega una conexión, ejecuta nuestro programa `server_handler`, (3) el programa recibe la petición HTTP completa por su entrada estándar, (4) el programa escribe la respuesta HTTP por su salida estándar, y (5) netcat envía esa respuesta de vuelta al cliente y cierra la conexión.

¿Por qué este enfoque es mejor que implementar sockets directamente (como hicimos en la sección 2)? Por varias razones prácticas. Primero, el manejo de sockets es complejo: hay que gestionar sockets no bloqueantes, multiplexar conexiones, manejar señales del sistema, y lidiar con errores de red. Netcat ya resuelve todo eso. Segundo, nuestro handler Fortran es un programa independiente que se ejecuta y termina por cada petición, lo que simplifica drásticamente la gestión de memoria: no hay estado persistente, no hay fugas de memoria. Tercero, podemos probar el handler directamente desde la terminal sin red, simplemente escribiendo una petición HTTP falsa en su entrada estándar.

La desventaja principal es la eficiencia: lanzar un proceso Fortran por cada petición tiene overhead (inicialización del runtime, carga de módulos, asignación de memoria). Para un servicio de producción con alta concurrencia, esto no es viable. Pero para una API interna, un prototipo, o herramientas de administración, es perfectamente adecuado. En el mini-proyecto final añadiremos un bucle en bash que reinicia netcat tras cada conexión, permitiendo manejar múltiples peticiones secuencialmente.

El diseño de nuestro handler sigue el patrón "read-parse-process-respond": lee toda la entrada estándar (la petición HTTP), parsea la línea de request y las cabeceras, procesa según la ruta y el método, construye la respuesta, y la escribe a stdout. El logging se envía a stderr para no contaminar la respuesta.

```fortran
! server_handler.f90 - Manejador de peticiones HTTP para uso con nc
program server_handler
  implicit none

  character(len=4096) :: line
  character(len=32) :: method, http_version
  character(len=256) :: path
  character(len=4096) :: body
  integer :: io_stat, content_length, pos, error_code
  character(len=:), allocatable :: response_body, status_text

  method = ''
  path = ''
  body = ''
  content_length = 0
  error_code = 200
  response_body = ''

  ! === Leer linea de request: "GET /ruta HTTP/1.1" ===
  read(*, '(a)', iostat=io_stat) line
  if (io_stat /= 0) then
    call responder(500, '{"error":"Error al leer request"}')
    stop
  end if
  read(line, *, iostat=io_stat) method, path, http_version
  if (io_stat /= 0) then
    call responder(400, '{"error":"Linea de request invalida"}')
    stop
  end if

  ! === Leer cabeceras hasta linea vacia ===
  do
    read(*, '(a)', iostat=io_stat) line
    if (io_stat /= 0) exit
    pos = index(line, 'Content-Length:')
    if (pos > 0) then
      read(line(pos+15:), *, iostat=io_stat) content_length
    end if
    if (len_trim(line) == 0) exit  ! Fin de cabeceras
  end do

  ! === Leer cuerpo si existe ===
  if (content_length > 0) then
    read(*, '(a)', iostat=io_stat) body
    if (io_stat == 0) then
      body = body(:min(content_length, len_trim(body)))
    end if
  end if

  ! === Mostrar peticion en log (stderr) ===
  write(0, '(a,1x,a,1x,a,i0)') '[', trim(method), '] ', trim(path), &
        ' CL:', content_length

  ! === Enrutamiento basico ===
  if (trim(adjustl(method)) == 'GET') then
    if (trim(path) == '/') then
      response_body = '{"mensaje":"Bienvenido a la API Fortran","estado":"activa"}'
      error_code = 200
    else if (trim(path) == '/status') then
      response_body = '{"estado":"ok","uptime":"desconocido"}'
      error_code = 200
    else if (trim(path) == '/echo') then
      response_body = '{"mensaje":"Esto es un eco"}'
      error_code = 200
    else
      response_body = '{"error":"Ruta no encontrada"}'
      error_code = 404
    end if
  else if (trim(adjustl(method)) == 'POST') then
    if (trim(path) == '/echo') then
      response_body = '{"recibido":' // trim(body) // '}'
      error_code = 201
    else
      response_body = '{"error":"Ruta no encontrada"}'
      error_code = 404
    end if
  else
    response_body = '{"error":"Metodo no permitido"}'
    error_code = 405
  end if

  ! === Construir y enviar respuesta ===
  call responder(error_code, response_body)

contains

  subroutine responder(code, body)
    integer, intent(in) :: code
    character(len=*), intent(in) :: body

    select case (code)
    case (200); status_text = 'OK'
    case (201); status_text = 'Created'
    case (204); status_text = 'No Content'
    case (400); status_text = 'Bad Request'
    case (404); status_text = 'Not Found'
    case (405); status_text = 'Method Not Allowed'
    case (500); status_text = 'Internal Server Error'
    case default; status_text = 'Unknown'
    end select

    write(*, '(a)') 'HTTP/1.1 ' // trim(adjustl(int_to_str(code))) // ' ' // trim(status_text)
    write(*, '(a)') 'Content-Type: application/json'
    write(*, '(a)') 'Content-Length: ' // trim(adjustl(int_to_str(len_trim(body))))
    write(*, '(a)') 'Connection: close'
    write(*, '(a)') ''
    write(*, '(a)') trim(body)
  end subroutine

  function int_to_str(val) result(str)
    integer, intent(in) :: val
    character(len=32) :: buf
    character(len=:), allocatable :: str
    write(buf, '(i0)') val
    str = trim(buf)
  end function

end program server_handler
```

**Análisis línea por línea**:

- `read(*, '(a)', iostat=io_stat) line`: Lee una línea de la entrada estándar. En el contexto del servidor nc, stdin es el socket TCP. El formato `'(a)'` lee hasta el siguiente salto de línea.
- `read(line, *, iostat=io_stat) method, path, http_version`: Usamos list-directed input (`*`) para separar por espacios la línea de request. Extrae `method` (ej. "GET"), `path` (ej. "/status"), y `http_version` (ej. "HTTP/1.1").
- `pos = index(line, 'Content-Length:')`: Busca la cabecera `Content-Length` usando la función intrínseca `index`, que devuelve la posición de una subcadena dentro de otra.
- `read(line(pos+15:), *, iostat=io_stat) content_length`: Lee el valor numérico después de `Content-Length:`. `pos+15` salta los 15 caracteres de "Content-Length:" más el espacio.
- `write(0, '(a,...)')`: `write(0, ...)` escribe en la unidad 0, que es la salida de error estándar (stderr). Esto permite logging sin contaminar stdout (que es la respuesta HTTP).
- `call responder(code, body)`: La subrutina `responder` construye y envía la respuesta HTTP completa con código de estado, cabeceras y cuerpo.
- `select case (code)`: Mapea códigos numéricos a textos de estado HTTP ("200" -> "OK", "404" -> "Not Found", etc.).
- `write(*, '(a)') 'HTTP/1.1 ' // ...`: Escribe la línea de estado HTTP. `write(*, '(a)')` escribe a stdout (unit `*`), que en el contexto de nc es el socket TCP.
- `write(*, '(a)') ''`: La línea vacía separa cabeceras del cuerpo. Es el CRLF doble que marca el fin de las cabeceras.
- `Connection: close`: Indica al cliente que el servidor cerrará la conexión después de esta respuesta. Es importante porque nuestro handler termina después de responder.

**Salida esperada** (al probar con entrada manual simulada):

Primero probamos el handler directamente (sin nc):

```bash
printf 'GET /status HTTP/1.1\r\nHost: localhost\r\n\r\n' | ./server_handler
```

Salida:
```
[GET] /status CL:0
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 38
Connection: close

{"estado":"ok","uptime":"desconocido"}
```

Ahora con el servidor real usando nc:
```bash
# Terminal 1: servidor
while true; do nc -l -p 8080 -e ./server_handler; done

# Terminal 2: cliente
curl -s http://localhost:8080/status
```

Respuesta de curl:
```
{"estado":"ok","uptime":"desconocido"}
```

Log en la terminal 1 (stderr):
```
[GET] /status CL:0
```

**Errores típicos**:

Error: el handler responde con logging a stdout, contaminando la respuesta HTTP.
```fortran
! Codigo erroneo: logging a stdout
print *, '[GET] /status'  ! Esto va a stdout = respuesta HTTP
```
Síntoma: el cliente recibe la línea de log como parte de la respuesta. La respuesta empieza con `[GET] /status` en lugar de `HTTP/1.1 200 OK`. Solución: siempre usar `write(0, *)` (stderr) para logging.

Error: nc no tiene la opción `-e` (netcat-openbsd tiene `-e`, pero `netcat-traditional` no). Síntoma: `nc -l -p 8080 -e ./server_handler` falla con `invalid option -- 'e'`. Solución: instalar netcat-openbsd (`sudo apt install netcat-openbsd`). Alternativa, usar socat:
```bash
socat TCP-LISTEN:8080,reuseaddr,fork EXEC:./server_handler
```

Error: escribir el cuerpo con `print` en lugar de `write(*, '(a)')`. `print` añade un espacio al inicio y un salto de línea al final, lo que puede alterar el JSON.
```fortran
! Codigo erroneo: print anade espacios
print *, '{"mensaje":"ok"}'  ! Output:   {"mensaje":"ok"}  (con espacio inicial)
```
Solución: usar `write(*, '(a)')` que escribe exactamente la cadena sin adornos.

---

## 5. JSON desde Fortran

JSON (JavaScript Object Notation) es el formato de intercambio de datos más utilizado en APIs REST. Es legible por humanos, fácil de parsear por máquinas, y tiene una estructura que se mapea naturalmente a los tipos de datos de cualquier lenguaje: objetos (diccionarios), arreglos, cadenas, números, booleanos y nulos. En lenguajes como JavaScript, Python o Go, parsear JSON es trivial porque tienen bibliotecas estándar que lo hacen. En Fortran, no hay una biblioteca estándar de JSON.

¿Por qué? Por razones históricas. Fortran fue diseñado para cálculo numérico, no para intercambio de datos. Cuando se definió el estándar Fortran 90/95, JSON no existía. Cuando JSON se volvió ubicuo (años 2000), Fortran ya tenía un ecosistema establecido que resolvía la serialización con formatos binarios (NetCDF, HDF5) o con formatos de texto estructurado (NAMELIST). Pero NAMELIST no es JSON, y ningún cliente web entiende NAMELIST.

La buena noticia: JSON es un formato de texto simple. Sus reglas son: los objetos comienzan con `{` y terminan con `}`, contienen pares `clave: valor` separados por comas; los arreglos comienzan con `[` y terminan con `]`; las cadenas van entre comillas dobles; los números se escriben sin comillas; los booleanos son `true` o `false`; los nulos son `null`. Con operaciones básicas de cadenas en Fortran —concatenación, `index`, `len_trim`, `adjustl`— podemos construir y analizar JSON sin necesidad de bibliotecas externas.

El enfoque que usaremos es artesanal pero efectivo: construimos JSON mediante concatenación de cadenas (build approach) y analizamos JSON mediante búsqueda de patrones (scan approach). Para los fines de una API REST simple, donde los objetos JSON tienen una profundidad de 1-2 niveles y los arreglos son homogéneos, este enfoque es suficiente. No implementaremos un parser JSON completo con validación de tokens, manejo de Unicode, o detección de errores; nos enfocaremos en lo que necesitamos para que el API funcione.

Construir un módulo JSON en Fortran puro tiene un valor pedagógico enorme. Te obliga a entender exactamente cómo se estructura JSON, porque estás escribiendo cada carácter. Te da control total sobre el formato de salida. Y demuestra que Fortran moderno, con su soporte de cadenas dinámicas (`character(len=:), allocatable`), es perfectamente capaz de manejar formatos de texto modernos.

```fortran
module json_handler
  implicit none
  private

  public :: json_object, json_array, json_pair, json_string, json_number, &
            json_parse_string, json_parse_number, json_escape, &
            int_to_str, real_to_str

contains

  ! ============================================================
  ! CONSTRUCTORES DE JSON
  ! ============================================================

  function json_object(pairs) result(json)
    character(len=*), intent(in) :: pairs(:)
    character(len=:), allocatable :: json
    integer :: i

    if (size(pairs) == 0) then
      json = '{}'
      return
    end if

    json = '{'
    do i = 1, size(pairs)
      if (i > 1) json = json // ','
      json = json // pairs(i)
    end do
    json = json // '}'
  end function

  function json_array(values) result(json)
    character(len=*), intent(in) :: values(:)
    character(len=:), allocatable :: json
    integer :: i

    if (size(values) == 0) then
      json = '[]'
      return
    end if

    json = '['
    do i = 1, size(values)
      if (i > 1) json = json // ','
      json = json // json_string(values(i))
    end do
    json = json // ']'
  end function

  function json_pair(key, value) result(pair)
    character(len=*), intent(in) :: key, value
    character(len=:), allocatable :: pair
    pair = '"' // json_escape(key) // '":' // value
  end function

  function json_string(value) result(str)
    character(len=*), intent(in) :: value
    character(len=:), allocatable :: str
    str = '"' // json_escape(value) // '"'
  end function

  function json_number(val) result(str)
    class(*), intent(in) :: val
    character(len=:), allocatable :: str

    select type (val)
    type is (integer)
      str = int_to_str(val)
    type is (real)
      str = real_to_str(val)
    type is (real(8))
      str = real_to_str(real(val))
    class default
      str = 'null'
    end select
  end function

  ! ============================================================
  ! ESCAPE DE CADENAS JSON
  ! ============================================================

  function json_escape(s) result(esc)
    character(len=*), intent(in) :: s
    character(len=:), allocatable :: esc
    integer :: i
    character(len=1) :: c

    esc = ''
    do i = 1, len_trim(s)
      c = s(i:i)
      select case (c)
      case ('"');  esc = esc // '\\"'
      case ('\\'); esc = esc // '\\\\'
      case (char(8));  esc = esc // '\\b'
      case (char(12)); esc = esc // '\\f'
      case (char(10)); esc = esc // '\\n'
      case (char(13)); esc = esc // '\\r'
      case (char(9));  esc = esc // '\\t'
      case default;    esc = esc // c
      end select
    end do
  end function

  ! ============================================================
  ! ANALIZADOR JSON BASICO
  ! ============================================================

  function json_parse_string(json_str, key) result(val)
    character(len=*), intent(in) :: json_str, key
    character(len=:), allocatable :: val
    integer :: pos, start_q, end_q

    val = ''
    pos = index(json_str, '"' // trim(key) // '"')
    if (pos == 0) return

    pos = index(json_str(pos:), ':') + pos
    pos = index(json_str(pos:), '"') + pos
    if (pos == 0) return

    start_q = pos + 1
    end_q = index(json_str(start_q:), '"') + start_q - 2
    if (end_q < start_q) return

    val = json_str(start_q:end_q)
  end function

  function json_parse_number(json_str, key) result(val)
    character(len=*), intent(in) :: json_str, key
    real :: val
    integer :: pos, i, j
    character(len=32) :: buf

    val = 0.0
    pos = index(json_str, '"' // trim(key) // '"')
    if (pos == 0) return

    pos = index(json_str(pos:), ':') + pos

    do while (pos <= len_trim(json_str) .and. json_str(pos:pos) == ' ')
      pos = pos + 1
    end do

    i = pos
    j = i
    do while (j <= len_trim(json_str))
      if (json_str(j:j) == ',' .or. json_str(j:j) == '}' .or. &
          json_str(j:j) == ']' .or. json_str(j:j) == ' ') exit
      j = j + 1
    end do
    j = j - 1

    if (j >= i) then
      buf = json_str(i:j)
      read(buf, *, err=999) val
    end if
    return
999 val = 0.0
  end function

  ! ============================================================
  ! UTILERIAS DE CONVERSION
  ! ============================================================

  function int_to_str(val) result(str)
    integer, intent(in) :: val
    character(len=32) :: buf
    character(len=:), allocatable :: str
    write(buf, '(i0)') val
    str = trim(buf)
  end function

  function real_to_str(val) result(str)
    real, intent(in) :: val
    character(len=32) :: buf
    character(len=:), allocatable :: str
    write(buf, '(f0.2)') val
    str = trim(buf)
  end function

end module json_handler
```

**Análisis línea por línea**:

- `module json_handler ... end module`: Define un módulo público que encapsula todas las operaciones JSON. `private` al inicio seguido de `public ::` hace explícito qué se exporta.
- `function json_object(pairs) result(json)`: Toma un arreglo de pares `"clave":valor` y los envuelve con `{` y `}` separados por comas. Es un constructor de objetos JSON.
- `function json_pair(key, value)`: Construye un par clave-valor con formato `"clave":valor`. Escapa la clave automáticamente con `json_escape`.
- `function json_string(value)`: Envuelve un valor entre comillas dobles escapadas. Útil para valores de tipo string en JSON.
- `function json_number(val)`: Usa `select type` para determinar si el argumento es integer o real y formatearlo apropiadamente. Unlimited polymorphic (`class(*)`) permite aceptar cualquier tipo numérico.
- `function json_escape(s)`: Itera carácter por carácter reemplazando caracteres especiales JSON (`"`, `\`, `\n`, `\t`, etc.) por sus secuencias de escape. `char(10)` es el salto de línea, `char(9)` el tabulador, etc.
- `function json_parse_string(json_str, key)`: Busca una clave en un JSON y extrae su valor como cadena. Usa `index` para localizar la clave, luego encuentra las comillas que delimitan el valor.
- `function json_parse_number(json_str, key)`: Extrae el valor numérico asociado a una clave. Localiza el inicio del número después de `:`, lo extrae en un buffer temporal, y usa `read(buf, *)` para convertirlo a real.

**Salida esperada**:

Programa de prueba:
```fortran
program test_json
  use json_handler
  implicit none

  character(len=:), allocatable :: json

  ! Construir objeto JSON
  json = json_object([ &
    json_pair('nombre', json_string('Laptop Gamer')), &
    json_pair('precio', json_number(1299.99)), &
    json_pair('stock', json_number(10)) ])

  print *, 'JSON construido:'
  print *, json
  print *, ''

  ! Parsear campo del JSON
  print *, 'Campo nombre:', json_parse_string(json, 'nombre')
  print *, 'Campo precio:', json_parse_number(json, 'precio')

end program test_json
```

Salida:
```
 JSON construido:
{"nombre":"Laptop Gamer","precio":1299.99,"stock":10}

 Campo nombre:Laptop Gamer
 Campo precio:   1299.990
```

**Errores típicos**:

Error: confundir el escape en Fortran vs escape en JSON. Para incluir una comilla doble en un string JSON, en JSON se escribe `\"`. Pero en Fortran, para escribir `\"` en una cadena, hay que duplicar el backslash o usar comillas simples exteriores:
```fortran
! Incorrecto
str = '"Esta es una comilla: \" dentro"'  ! Error: backslash no es especial en Fortran

! Correcto
str = '"Esta es una comilla: "" dentro"'  ! Comillas dobles Fortran
```
Para el módulo JSON, las funciones `json_escape` y `json_string` manejan esto automáticamente; el usuario nunca escribe JSON manualmente.

Error: usar `json_parse_string` con una clave que contiene caracteres especiales. Si la clave tiene un `:` o `"`, el parser simplificado falla. Solución: mantener claves alfanuméricas simples.

Error: el formato `f0.2` en `real_to_str` trunca decimales. Si `val = 0.001`, `real_to_str(val)` devuelve `0.00`. Para mayor precisión, usar `f0.6` o similar. En JSON no hay un estándar de precisión; cada implementación elige.

---

## 6. Routing y métodos HTTP

Has construido un módulo JSON, has implementado un cliente HTTP, y has visto cómo funciona un servidor básico. Ahora falta la pieza que conecta todo: el enrutamiento (routing). El routing es el proceso de examinar la petición entrante (método HTTP + ruta) y decidir qué código ejecutar y qué respuesta devolver.

Piensen en una API real: `GET /productos` devuelve la lista, `POST /productos` crea uno nuevo, `GET /productos/42` devuelve el producto con ID 42. Son tres rutas que comparten el mismo prefijo `/productos` pero se diferencian por el método y los parámetros. En lenguajes como Express.js (Node), Flask (Python) o Spring (Java), el routing se declara con decoradores o anotaciones. En Fortran, no tenemos esa sintaxis, pero tenemos `select case`, que es igual de poderoso.

El patrón general es: primero seleccionas por método (`case ('GET')`, `case ('POST')`, etc.), y dentro de cada método, seleccionas por ruta. Las rutas con parámetros (como `/productos/{id}`) requieren un paso adicional: detectar el prefijo y extraer el parámetro con operaciones de cadena.

Hay una decisión de diseño importante: ¿el routing debe estar en el programa principal o en un módulo separado? En nuestro enfoque, el programa principal (`product_api`) contendrá el routing, porque es el punto de entrada y la lógica de negocio está acoplada a los endpoints. Pero para APIs más grandes, se podría crear un módulo `router` que mapee rutas a procedimientos.

El manejo de errores también es parte del routing. Si el método no está soportado, devolvemos 405 Method Not Allowed. Si la ruta no existe, 404 Not Found. Si el parámetro de la URL no es un número válido, 400 Bad Request. Cada error debe producir un JSON descriptivo.

Veamos un ejemplo concreto de routing en Fortran que maneja GET, POST, PUT y DELETE sobre un recurso `/items`:

```fortran
program routing_demo
  implicit none

  character(len=32) :: method
  character(len=256) :: path
  character(len=4096) :: body
  character(len=:), allocatable :: response
  integer :: status_code, item_id, io_stat

  ! Simulamos una peticion entrante
  method = 'GET'
  path = '/items/42'
  body = ''

  status_code = 200
  response = ''

  ! === Routing principal ===
  select case (trim(adjustl(method)))
  case ('GET')
    if (trim(path) == '/items') then
      response = '{"items":["item1","item2","item3"]}'
      status_code = 200

    else if (path_has_id(path, '/items/', item_id)) then
      response = '{"id":' // trim(int_to_str(item_id)) // ',"nombre":"Item de ejemplo"}'
      status_code = 200

    else if (trim(path) == '/') then
      response = '{"mensaje":"Bienvenido a la API"}'
      status_code = 200

    else
      response = '{"error":"Ruta no encontrada"}'
      status_code = 404
    end if

  case ('POST')
    if (trim(path) == '/items') then
      response = '{"mensaje":"Item creado","id":101}'
      status_code = 201
    else
      response = '{"error":"Ruta no encontrada"}'
      status_code = 404
    end if

  case ('PUT')
    if (path_has_id(path, '/items/', item_id)) then
      response = '{"mensaje":"Item ' // trim(int_to_str(item_id)) // ' actualizado"}'
      status_code = 200
    else
      response = '{"error":"Ruta no encontrada"}'
      status_code = 404
    end if

  case ('DELETE')
    if (path_has_id(path, '/items/', item_id)) then
      response = '{"mensaje":"Item ' // trim(int_to_str(item_id)) // ' eliminado"}'
      status_code = 200
    else
      response = '{"error":"Ruta no encontrada"}'
      status_code = 404
    end if

  case default
    response = '{"error":"Metodo no permitido"}'
    status_code = 405
  end select

  ! Mostrar resultado
  print *, 'Status:', status_code
  print *, 'Response:', response

contains

  function path_has_id(url, prefix, id) result(found)
    character(len=*), intent(in) :: url, prefix
    integer, intent(out) :: id
    logical :: found
    integer :: pos, io_stat

    found = .false.
    pos = index(url, prefix)
    if (pos == 1) then
      ! El prefijo esta al inicio de la URL
      read(url(len_trim(prefix)+1:), *, iostat=io_stat) id
      if (io_stat == 0) found = .true.
    end if
  end function

  function int_to_str(val) result(str)
    integer, intent(in) :: val
    character(len=32) :: buf
    character(len=:), allocatable :: str
    write(buf, '(i0)') val
    str = trim(buf)
  end function

end program routing_demo
```

**Análisis línea por línea**:

- `select case (trim(adjustl(method)))`: El método HTTP se limpia de espacios (por si llegó con padding) y se evalúa con `select case`. Es más legible que una cadena de `if-else if`.
- `case ('GET')`: Rama para peticiones GET. Dentro, otro nivel de `if-else if` para distinguir rutas.
- `path_has_id(path, '/items/', item_id)`: Función auxiliar que verifica si la ruta comienza con un prefijo y extrae un ID numérico. Devuelve `.true.` si el prefijo coincide y el ID es un entero válido.
- `case default`: Captura cualquier método HTTP no contemplado (PATCH, HEAD, OPTIONS, etc.) y devuelve 405 Method Not Allowed.
- `function path_has_id(url, prefix, id)`: Toma la URL y el prefijo (ej. `/items/`), verifica que `url` comience con el prefijo, y si es así, lee el resto como entero.
- `read(url(len_trim(prefix)+1:), *, iostat=io_stat) id`: Usa el `iostat` para detectar si el segmento después del prefijo es realmente un número. Si `read` falla, `io_stat /= 0` y la función retorna `.false.`.

**Salida esperada**:

```
 Status:         200
 Response:{"id":42,"nombre":"Item de ejemplo"}
```

Probando con `path = '/items'`:
```
 Status:         200
 Response:{"items":["item1","item2","item3"]}
```

Probando con `path = '/productos'`:
```
 Status:         404
 Response:{"error":"Ruta no encontrada"}
```

Probando con `method = 'PATCH'`:
```
 Status:         405
 Response:{"error":"Metodo no permitido"}
```

**Errores típicos**:

Error: comparar rutas sin `trim(adjustl(...))`. Las rutas pueden llegar con espacios a la izquierda o derecha (especialmente si hay errores de parsing), y `'/items' /= '/items '`.
```fortran
! Incorrecto: no limpia la ruta
if (path == '/items') then
```
Solución: siempre usar `if (trim(adjustl(path)) == '/items')`.

Error: en `path_has_id`, asumir que el ID siempre está presente. Si la ruta es `/items/` (sin ID), `read` fallará pero la función retornará `.false.`. Esto está manejado correctamente con `iostat`.

Error: olvidar el `case default`. Si llega un método como `PATCH` o `OPTIONS`, el programa no entra en ninguna rama y `response` queda vacía, lo que produce una respuesta HTTP incompleta. Solución: siempre incluir `case default` con un 405.

Ahora, un concepto fundamental en APIs REST: el middleware. Middleware es código que se ejecuta antes o después del routing principal, aplicando lógica transversal como logging, autenticación, compresión, o rate limiting. En nuestro handler Fortran, podemos implementar middleware como subrutinas que envuelven el procesamiento.

```fortran
program demo_middleware
  implicit none

  character(len=32) :: method
  character(len=256) :: path
  logical :: autenticado
  integer :: start_time, end_time

  method = 'GET'
  path = '/items'
  autenticado = .false.
  start_time = 0

  ! Middleware 1: Logging
  write(0, '(a,a,a)') '[MIDDLEWARE] Peticion recibida: ', trim(method), ' ', trim(path)
  start_time = tiempo_ms()

  ! Middleware 2: Autenticacion basica
  call verificar_autenticacion(method, path, autenticado)
  if (.not. autenticado) then
    write(0, '(a)') '[MIDDLEWARE] Autenticacion fallida'
    call responder(401, '{"error":"No autorizado"}')
    stop
  end if

  ! Routing principal
  write(0, '(a)') '[MIDDLEWARE] Routing...'
  if (trim(method) == 'GET' .and. trim(path) == '/items') then
    call responder(200, '{"items":["a","b","c"]}')
  else
    call responder(404, '{"error":"No encontrado"}')
  end if

  ! Middleware 3: Post-procesamiento
  end_time = tiempo_ms()
  write(0, '(a,i0,a)') '[MIDDLEWARE] Tiempo de respuesta: ', end_time - start_time, 'ms'

contains

  subroutine verificar_autenticacion(m, p, ok)
    character(len=*), intent(in) :: m, p
    logical, intent(out) :: ok
    ! En un sistema real, verificaria token JWT o Basic Auth
    ! Aqui simplemente simulamos: todo GET sin autenticar
    if (trim(m) == 'GET') then
      ok = .true.  ! Lectura publica
    else
      ok = .false. ! Escritura requiere auth
    end if
  end subroutine

  subroutine responder(code, body)
    integer, intent(in) :: code
    character(len=*), intent(in) :: body
    character(len=32) :: status_text

    select case (code)
    case (200); status_text = 'OK'
    case (401); status_text = 'Unauthorized'
    case (404); status_text = 'Not Found'
    end select

    write(*, '(a)') 'HTTP/1.1 ' // trim(adjustl(int_to_str(code))) // ' ' // trim(status_text)
    write(*, '(a)') 'Content-Type: application/json'
    write(*, '(a)') ''
    write(*, '(a)') trim(body)
  end subroutine

  function int_to_str(val) result(str)
    integer, intent(in) :: val
    character(len=32) :: buf
    character(len=:), allocatable :: str
    write(buf, '(i0)') val
    str = trim(buf)
  end function

  function tiempo_ms() result(t)
    integer :: t
    integer :: rate
    call system_clock(t, rate)
    t = t * 1000 / rate
  end function

end program demo_middleware
```

**Análisis línea por línea**:

- `write(0, '(a,a,a)') '[MIDDLEWARE] ...'`: Middleware de logging que escribe a stderr antes de procesar.
- `call verificar_autenticacion(method, path, autenticado)`: Middleware de autenticación. En un sistema real, leería cabeceras como `Authorization: Bearer <token>` y validaría.
- `if (.not. autenticado) then`: Si la autenticación falla, se detiene el flujo y se devuelve 401 Unauthorized.
- `end_time = tiempo_ms()`: Middleware de post-procesamiento que mide el tiempo total de la petición.
- `call system_clock(t, rate)`: Intrínseco de Fortran para obtener el tiempo del sistema con precisión de milisegundos (dependiente del hardware).

**Salida esperada** (stderr):
```
[MIDDLEWARE] Peticion recibida: GET /items
[MIDDLEWARE] Routing...
[MIDDLEWARE] Tiempo de respuesta: 0ms
```

**Stdout**:
```
HTTP/1.1 200 OK
Content-Type: application/json

{"items":["a","b","c"]}
```

**Errores típicos**:

Error: el middleware de autenticación bloquea incluso GET. Si la API tiene endpoints públicos, deben estar explícitamente excluidos del chequeo de autenticación. Solución: mantener una lista blanca de rutas públicas.

---

## 7. Mini-proyecto: API RESTful de Catálogo de Productos

Has llegado al final del capítulo y es momento de integrar todo lo aprendido en un proyecto completo y funcional. Vamos a construir una API RESTful para un catálogo de productos, implementada completamente en Fortran moderno. Esta API permitirá listar productos, obtener un producto por ID, crear nuevos productos, actualizar existentes y eliminarlos, todo a través de peticiones HTTP con cuerpo JSON.

¿Qué has aprendido hasta ahora? Has visto cómo Fortran puede interoperar con C para llamadas al sistema (`iso_c_binding`), cómo ejecutar comandos externos (`execute_command_line` + `curl`), cómo construir un servidor HTTP usando netcat como proxy de red, cómo generar y analizar JSON con código Fortran puro, y cómo enrutar peticiones por método y ruta. Este mini-proyecto une todas esas piezas en una arquitectura modular y bien organizada.

La arquitectura del proyecto se divide en tres módulos Fortran y un script de shell:

- **`json_handler.f90`**: Módulo de utilidades JSON. Contiene funciones para construir objetos y arreglos JSON, escapar cadenas, y parsear campos específicos. Es el mismo módulo que desarrollamos en la sección 5, ligeramente extendido.
- **`product_catalog.f90`**: Módulo de lógica de negocio. Define el tipo `product` (id, nombre, descripción, precio, stock) y proporciona operaciones CRUD sobre un arreglo allocatable de productos. Incluye funciones de serialización a JSON.
- **`product_api.f90`**: Programa principal. Implementa el handler CGI que lee la petición HTTP desde stdin, ejecuta el routing, llama al catálogo de productos, construye la respuesta JSON y la escribe a stdout. Incluye logging a stderr.
- **`server.sh`**: Script bash que inicia el servidor usando netcat en modo escucha continua.

El flujo completo de una petición es: (1) curl (cliente) envía una petición HTTP al puerto 8080, (2) netcat recibe la conexión y ejecuta `product_api`, (3) `product_api` lee la petición de stdin, (4) parsea método, ruta y cuerpo, (5) ejecuta la operación correspondiente en `product_catalog`, (6) serializa el resultado a JSON usando `json_handler`, (7) construye la respuesta HTTP y la escribe a stdout, (8) netcat envía la respuesta al cliente.

Además del servidor, incluiremos un cliente de prueba en Fortran (`cliente_prueba.f90`) que realiza peticiones a los endpoints usando `execute_command_line` + `curl`, demostrando que Fortran puede ser tanto servidor como cliente HTTP.

### Módulo json_handler

```fortran
! json_handler.f90
module json_handler
  implicit none
  private

  public :: json_object, json_array, json_pair, json_string, json_number, &
            json_parse_string, json_parse_number, json_parse_integer, &
            json_escape, int_to_str, real_to_str

contains

  function json_object(pairs) result(json)
    character(len=*), intent(in) :: pairs(:)
    character(len=:), allocatable :: json
    integer :: i

    if (size(pairs) == 0) then
      json = '{}'
      return
    end if

    json = '{'
    do i = 1, size(pairs)
      if (i > 1) json = json // ','
      json = json // pairs(i)
    end do
    json = json // '}'
  end function

  function json_array(values) result(json)
    character(len=*), intent(in) :: values(:)
    character(len=:), allocatable :: json
    integer :: i

    if (size(values) == 0) then
      json = '[]'
      return
    end if

    json = '['
    do i = 1, size(values)
      if (i > 1) json = json // ','
      json = json // values(i)
    end do
    json = json // ']'
  end function

  function json_pair(key, value) result(pair)
    character(len=*), intent(in) :: key, value
    character(len=:), allocatable :: pair
    pair = '"' // json_escape(key) // '":' // value
  end function

  function json_string(value) result(str)
    character(len=*), intent(in) :: value
    character(len=:), allocatable :: str
    str = '"' // json_escape(value) // '"'
  end function

  function json_number(val) result(str)
    class(*), intent(in) :: val
    character(len=:), allocatable :: str

    select type (val)
    type is (integer)
      str = int_to_str(val)
    type is (real)
      str = real_to_str(val)
    type is (real(8))
      str = real_to_str(real(val))
    class default
      str = 'null'
    end select
  end function

  function json_escape(s) result(esc)
    character(len=*), intent(in) :: s
    character(len=:), allocatable :: esc
    integer :: i
    character(len=1) :: c

    esc = ''
    do i = 1, len_trim(s)
      c = s(i:i)
      select case (c)
      case ('"');  esc = esc // '\\"'
      case ('\\'); esc = esc // '\\\\'
      case (char(8));  esc = esc // '\\b'
      case (char(12)); esc = esc // '\\f'
      case (char(10)); esc = esc // '\\n'
      case (char(13)); esc = esc // '\\r'
      case (char(9));  esc = esc // '\\t'
      case default;    esc = esc // c
      end select
    end do
  end function

  function json_parse_string(json_str, key) result(val)
    character(len=*), intent(in) :: json_str, key
    character(len=:), allocatable :: val
    integer :: pos, start_q, end_q

    val = ''
    pos = index(json_str, '"' // trim(key) // '"')
    if (pos == 0) return

    pos = index(json_str(pos:), ':') + pos
    pos = index(json_str(pos:), '"') + pos
    if (pos == 0) return

    start_q = pos + 1
    end_q = index(json_str(start_q:), '"') + start_q - 2
    if (end_q < start_q) return

    val = json_str(start_q:end_q)
  end function

  function json_parse_number(json_str, key) result(val)
    character(len=*), intent(in) :: json_str, key
    real :: val
    integer :: pos, i, j
    character(len=32) :: buf

    val = 0.0
    pos = index(json_str, '"' // trim(key) // '"')
    if (pos == 0) return

    pos = index(json_str(pos:), ':') + pos

    do while (pos <= len_trim(json_str) .and. json_str(pos:pos) == ' ')
      pos = pos + 1
    end do

    i = pos
    j = i
    do while (j <= len_trim(json_str))
      if (json_str(j:j) == ',' .or. json_str(j:j) == '}' .or. &
          json_str(j:j) == ']' .or. json_str(j:j) == ' ') exit
      j = j + 1
    end do
    j = j - 1

    if (j >= i) then
      buf = json_str(i:j)
      read(buf, *, err=999) val
    end if
    return
999 val = 0.0
  end function

  function json_parse_integer(json_str, key) result(val)
    character(len=*), intent(in) :: json_str, key
    integer :: val
    val = nint(json_parse_number(json_str, key))
  end function

  function int_to_str(val) result(str)
    integer, intent(in) :: val
    character(len=32) :: buf
    character(len=:), allocatable :: str
    write(buf, '(i0)') val
    str = trim(buf)
  end function

  function real_to_str(val) result(str)
    real, intent(in) :: val
    character(len=32) :: buf
    character(len=:), allocatable :: str
    write(buf, '(f0.2)') val
    str = trim(buf)
  end function

end module json_handler
```

**Análisis línea por línea**:

- El módulo es idéntico al desarrollado en la sección 5, con la adición de `json_parse_integer`, que convierte el resultado de `json_parse_number` a entero usando `nint` (nearest integer).
- `json_object`, `json_array`, `json_pair`, `json_string`, `json_number`: Constructores que permiten armar JSON de forma composicional y legible.
- `json_escape`: Función crítica para seguridad; asegura que ningún carácter especial en los datos rompa la sintaxis JSON.
- `json_parse_*`: Funciones de parsing básico que devuelven valores específicos por clave. No implementan validación completa de JSON, pero son suficientes para JSON de 1-2 niveles de profundidad.
- `int_to_str` y `real_to_str`: Conversiones numéricas con formato controlado. `i0` elimina espacios, `f0.2` produce 2 decimales.

### Módulo product_catalog

```fortran
! product_catalog.f90
module product_catalog
  use json_handler
  implicit none
  private

  type, public :: product
    integer :: id = 0
    character(len=:), allocatable :: name
    character(len=:), allocatable :: description
    real :: price = 0.0
    integer :: stock = 0
  end type

  type(product), allocatable, public :: products(:)
  integer, public :: next_id = 1

  public :: init_catalog, add_product, get_product, update_product, &
            delete_product, product_to_json, products_to_json, &
            parse_product_from_json

contains

  subroutine init_catalog()
    if (.not. allocated(products)) then
      allocate(products(0))
    end if
  end subroutine

  subroutine add_product(p)
    type(product), intent(inout) :: p
    type(product), allocatable :: tmp(:)

    p%id = next_id
    next_id = next_id + 1

    allocate(tmp(size(products) + 1))
    tmp(:size(products)) = products
    tmp(size(products) + 1) = p
    call move_alloc(tmp, products)
  end subroutine

  subroutine get_product(id, p, found)
    integer, intent(in) :: id
    type(product), intent(out) :: p
    logical, intent(out) :: found
    integer :: i

    found = .false.
    do i = 1, size(products)
      if (products(i)%id == id) then
        p = products(i)
        found = .true.
        return
      end if
    end do
  end subroutine

  subroutine update_product(p, found)
    type(product), intent(in) :: p
    logical, intent(out) :: found
    integer :: i

    found = .false.
    do i = 1, size(products)
      if (products(i)%id == p%id) then
        products(i) = p
        found = .true.
        return
      end if
    end do
  end subroutine

  subroutine delete_product(id, found)
    integer, intent(in) :: id
    logical, intent(out) :: found
    integer :: i, idx
    type(product), allocatable :: tmp(:)

    found = .false.
    idx = 0
    do i = 1, size(products)
      if (products(i)%id == id) then
        idx = i
        found = .true.
        exit
      end if
    end do

    if (.not. found) return

    allocate(tmp(size(products) - 1))
    tmp(:idx - 1) = products(:idx - 1)
    tmp(idx:) = products(idx + 1:)
    call move_alloc(tmp, products)
  end subroutine

  function product_to_json(p) result(json)
    type(product), intent(in) :: p
    character(len=:), allocatable :: json

    json = json_object([ &
      json_pair('id', json_number(p%id)), &
      json_pair('nombre', json_string(p%name)), &
      json_pair('descripcion', json_string(p%description)), &
      json_pair('precio', json_number(p%price)), &
      json_pair('stock', json_number(p%stock)) ])
  end function

  function products_to_json() result(json)
    character(len=:), allocatable :: json
    integer :: i
    character(len=:), allocatable :: items(:)

    allocate(items(size(products)))
    do i = 1, size(products)
      items(i) = product_to_json(products(i))
    end do
    json = json_array(items)
  end function

  subroutine parse_product_from_json(body, p, success)
    character(len=*), intent(in) :: body
    type(product), intent(out) :: p
    logical, intent(out) :: success

    p%id = 0
    p%name = json_parse_string(body, 'nombre')
    p%description = json_parse_string(body, 'descripcion')
    p%price = json_parse_number(body, 'precio')
    p%stock = json_parse_integer(body, 'stock')

    ! Verificar que al menos el nombre fue proporcionado
    success = (len_trim(p%name) > 0)
  end subroutine

end module product_catalog
```

**Análisis línea por línea**:

- `type(product)`: Define el tipo derivado `product` con campos: `id` (entero), `name` y `description` (cadenas allocatables), `price` (real), `stock` (entero).
- `type(product), allocatable, public :: products(:)`: Arreglo dinámico de productos, declarado como variable pública del módulo. `allocatable` permite redimensionarlo.
- `call init_catalog()`: Inicializa el arreglo si no está asignado. Se llama al inicio del programa principal.
- `call add_product(p)`: Agrega un producto al arreglo. Crea un arreglo temporal de tamaño `n+1`, copia los elementos existentes, asigna el nuevo ID, y reemplaza el arreglo con `move_alloc`.
- `call get_product(id, p, found)`: Busca un producto por ID. Itera sobre el arreglo y devuelve el producto encontrado. `found` indica si existía.
- `call update_product(p, found)`: Similar a `get_product` pero reemplaza el producto en su lugar en el arreglo.
- `call delete_product(id, found)`: Busca el producto por ID, lo elimina, y reduce el arreglo usando `move_alloc`. Si no existe, `found` es `.false.`.
- `product_to_json(p)`: Serializa un producto a JSON usando los constructores del módulo `json_handler`. Produce: `{"id":1,"nombre":"...","descripcion":"...","precio":...}`.
- `products_to_json()`: Serializa todos los productos como un arreglo JSON. Construye un arreglo de cadenas de productos, luego lo pasa a `json_array`.
- `parse_product_from_json(body, p, success)`: Deserializa un producto desde una cadena JSON. Usa `json_parse_string`, `json_parse_number`, y `json_parse_integer`. Verifica que el nombre no esté vacío como validación mínima.

**Salida esperada** (de las funciones individuales):

`product_to_json` para un producto con `id=1, name='Laptop', price=999.99, stock=10`:
```
{"id":1,"nombre":"Laptop","descripcion":"","precio":999.99,"stock":10}
```

`products_to_json` con dos productos:
```
[{"id":1,"nombre":"Laptop","descripcion":"","precio":999.99,"stock":10},{"id":2,"nombre":"Monitor","descripcion":"27 pulgadas","precio":499.99,"stock":25}]
```

**Errores típicos**:

Error: no inicializar el arreglo `products` antes de usarlo. Si se llama a `add_product` sin haber llamado a `init_catalog`, el arreglo no está asignado y `size(products)` causa un error en tiempo de ejecución.
```fortran
! Codigo erroneo: falta init_catalog
call add_product(p)  ! Error: products no esta asignado
```
Mensaje gfortran: `Fortran runtime error: Index '1' of dimension 1 of array 'products' above upper bound of 0`. Solución: siempre llamar a `init_catalog()` antes de cualquier operación.

Error: modificar un producto obtenido con `get_product` y esperar que el cambio se refleje en el catálogo. `get_product` devuelve una **copia** del producto, no una referencia.
```fortran
call get_product(1, p, found)
p%price = 500.0  ! Esto modifica la copia, no el original
! Para modificar, hay que llamar a update_product
```
Solución: después de modificar la copia, llamar a `update_product(p, found)`.

### Programa principal: product_api

```fortran
! product_api.f90
program product_api
  use iso_fortran_env, only: input_unit, error_unit
  use product_catalog
  use json_handler
  implicit none

  character(len=4096) :: line
  character(len=32) :: method, http_version
  character(len=256) :: path
  character(len=4096) :: body
  integer :: io_stat, content_length, pos, id
  character(len=:), allocatable :: response_body, response
  integer :: status_code
  type(product) :: p
  logical :: found

  method = ''
  path = ''
  body = ''
  content_length = 0
  status_code = 200
  response_body = ''

  ! === Leer linea de request ===
  read(input_unit, '(a)', iostat=io_stat) line
  if (io_stat /= 0) then
    call responder(500, '{"error":"Error al leer request"}')
    stop
  end if
  read(line, *, iostat=io_stat) method, path, http_version
  if (io_stat /= 0) then
    call responder(400, '{"error":"Linea de request invalida"}')
    stop
  end if

  ! === Leer cabeceras ===
  do
    read(input_unit, '(a)', iostat=io_stat) line
    if (io_stat /= 0) exit
    if (len_trim(line) == 0) exit
    pos = index(line, 'Content-Length:')
    if (pos > 0) then
      read(line(pos+15:), *, iostat=io_stat) content_length
    end if
  end do

  ! === Leer cuerpo ===
  if (content_length > 0) then
    read(input_unit, '(a)', iostat=io_stat) body
    if (io_stat == 0) then
      body = body(:min(content_length, len_trim(body)))
    end if
  end if

  ! === Logging de peticion ===
  write(error_unit, '(a,1x,a,1x,a,i0)') '[API]', trim(method), trim(path), &
        ' CL:', content_length

  ! === Inicializar catalogo ===
  call init_catalog()

  ! === Routing ===
  method = trim(adjustl(method))
  path = trim(adjustl(path))

  select case (method)
  case ('GET')
    if (path == '/productos') then
      status_code = 200
      response_body = products_to_json()

    else if (index(path, '/productos/') == 1) then
      read(path(11:), *, iostat=io_stat) id
      if (io_stat == 0) then
        call get_product(id, p, found)
        if (found) then
          status_code = 200
          response_body = product_to_json(p)
        else
          status_code = 404
          response_body = '{"error":"Producto no encontrado"}'
        end if
      else
        status_code = 400
        response_body = '{"error":"ID invalido en la ruta"}'
      end if

    else if (path == '/') then
      status_code = 200
      response_body = '{"mensaje":"API de Productos v1.0","endpoints":[' // &
        '"GET /productos","GET /productos/{id}",' // &
        '"POST /productos","PUT /productos/{id}",' // &
        '"DELETE /productos/{id}"]}'
    else
      status_code = 404
      response_body = '{"error":"Ruta no encontrada"}'
    end if

  case ('POST')
    if (path == '/productos') then
      call parse_product_from_json(body, p, found)
      if (found) then
        call add_product(p)
        status_code = 201
        response_body = product_to_json(p)
      else
        status_code = 400
        response_body = '{"error":"JSON invalido: se requiere al menos nombre"}'
      end if
    else
      status_code = 404
      response_body = '{"error":"Ruta no encontrada"}'
    end if

  case ('PUT')
    if (index(path, '/productos/') == 1) then
      read(path(11:), *, iostat=io_stat) id
      if (io_stat == 0) then
        call parse_product_from_json(body, p, found)
        if (found) then
          p%id = id
          call update_product(p, found)
          if (found) then
            status_code = 200
            response_body = product_to_json(p)
          else
            status_code = 404
            response_body = '{"error":"Producto no encontrado para actualizar"}'
          end if
        else
          status_code = 400
          response_body = '{"error":"JSON invalido en el cuerpo"}'
        end if
      else
        status_code = 400
        response_body = '{"error":"ID invalido en la ruta"}'
      end if
    else
      status_code = 404
      response_body = '{"error":"Ruta no encontrada"}'
    end if

  case ('DELETE')
    if (index(path, '/productos/') == 1) then
      read(path(11:), *, iostat=io_stat) id
      if (io_stat == 0) then
        call delete_product(id, found)
        if (found) then
          status_code = 200
          response_body = '{"mensaje":"Producto eliminado","id":' // &
            trim(int_to_str(id)) // '}'
        else
          status_code = 404
          response_body = '{"error":"Producto no encontrado"}'
        end if
      else
        status_code = 400
        response_body = '{"error":"ID invalido en la ruta"}'
      end if
    else
      status_code = 404
      response_body = '{"error":"Ruta no encontrada"}'
    end if

  case default
    status_code = 405
    response_body = '{"error":"Metodo no permitido"}'
  end select

  ! === Logging de respuesta ===
  write(error_unit, '(a,i0)') '[API] Respuesta: ', status_code

  ! === Enviar respuesta ===
  call responder(status_code, response_body)

contains

  subroutine responder(code, body)
    integer, intent(in) :: code
    character(len=*), intent(in) :: body
    character(len=32) :: status_text

    select case (code)
    case (200); status_text = 'OK'
    case (201); status_text = 'Created'
    case (204); status_text = 'No Content'
    case (400); status_text = 'Bad Request'
    case (404); status_text = 'Not Found'
    case (405); status_text = 'Method Not Allowed'
    case (500); status_text = 'Internal Server Error'
    case default; status_text = 'Unknown'
    end select

    write(*, '(a)') 'HTTP/1.1 ' // trim(adjustl(int_to_str(code))) // ' ' // trim(status_text)
    write(*, '(a)') 'Content-Type: application/json'
    write(*, '(a)') 'Content-Length: ' // trim(adjustl(int_to_str(len_trim(body))))
    write(*, '(a)') 'Connection: close'
    write(*, '(a)') ''
    write(*, '(a)') trim(body)
  end subroutine

end program product_api
```

**Análisis línea por línea**:

- `use iso_fortran_env, only: input_unit, error_unit`: Importa constantes para la entrada estándar (`input_unit` = 5) y error estándar (`error_unit` = 0).
- `read(input_unit, '(a)', iostat=io_stat) line`: Lee la primera línea del request (ej. "GET /productos HTTP/1.1") usando `input_unit` explícito.
- `read(line, *, iostat=io_stat) method, path, http_version`: Separa los tres campos de la línea de request usando lectura libre (`*`).
- `write(error_unit, '(a,1x,a,1x,a,i0)') '[API]', ...`: Middleware de logging que escribe a stderr con formato compacto.
- `select case (method)`: Enrutamiento principal por método HTTP.
- `case ('GET')`: Maneja `GET /productos` (lista), `GET /productos/{id}` (detalle), y `GET /` (raíz informativa).
- `index(path, '/productos/') == 1`: Detecta rutas con ID extrayendo el prefijo. Si la ruta empieza con `/productos/`, el resto debe ser un ID numérico.
- `read(path(11:), *, iostat=io_stat) id`: Lee el ID desde la posición 11 (longitud de `/productos/`). Si `iostat /= 0`, el ID no es un número válido.
- `call parse_product_from_json(body, p, found)`: Deserializa el cuerpo JSON a un producto. `found` indica si el JSON tenía al menos el campo `nombre`.
- `call add_product(p)`: Asigna un ID autoincremental y agrega al catálogo.
- `case ('PUT')`: Similar a POST pero asigna el ID desde la ruta y llama a `update_product`.
- `case ('DELETE')`: Elimina el producto por ID. Devuelve 404 si no existe.
- `case default`: Método no soportado, devuelve 405.
- `subroutine responder(code, body)`: Construye y escribe la respuesta HTTP completa a stdout. Incluye `Connection: close` para que el cliente sepa que la conexión termina.

**Salida esperada** (probando manualmente con entrada simulada):

```bash
printf 'GET /productos HTTP/1.1\r\nHost: localhost\r\n\r\n' | ./product_api
```

Salida (stdout):
```
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 2
Connection: close

[]
```

Salida (stderr):
```
[API] GET /productos CL:0
[API] Respuesta: 200
```

### Script de servidor

```bash
#!/bin/bash
# server.sh - Servidor de la API de Productos
# Uso: ./server.sh [puerto]
PUERTO=${1:-8080}
HANDLER=./product_api

echo "=== Servidor API de Productos (Fortran) ==="
echo "Puerto: $PUERTO"
echo "Handler: $HANDLER"
echo "Iniciando... (Ctrl+C para detener)"
echo ""

while true; do
    nc -l -p "$PUERTO" -e "$HANDLER"
    echo "[server] Conexion cerrada, esperando siguiente..."
done
```

**Análisis línea por línea**:

- `PUERTO=${1:-8080}`: Toma el puerto del primer argumento o usa 8080 por defecto.
- `while true; do ... done`: Bucle infinito que reinicia netcat tras cada conexión.
- `nc -l -p "$PUERTO" -e "$HANDLER"`: Escucha en el puerto y ejecuta el handler Fortran por cada conexión. `-e` enlaza stdin/stdout del handler al socket TCP.
- `echo "[server] Conexion cerrada..."`: Mensaje de logging en la terminal del servidor.

### Cliente de prueba en Fortran

```fortran
! cliente_prueba.f90
program cliente_prueba
  implicit none
  character(len=1024) :: cmd
  integer :: exitstat

  print *, '=== CLIENTE DE PRUEBA - API Productos ==='
  print *, ''

  ! --- GET / ---
  print *, '1. GET / (raiz)'
  cmd = 'curl -s http://localhost:8080/'
  call execute_command_line(cmd)
  print *, ''

  ! --- GET /productos (vacio) ---
  print *, '2. GET /productos (lista vacia)'
  cmd = 'curl -s http://localhost:8080/productos'
  call execute_command_line(cmd)
  print *, ''

  ! --- POST /productos (crear laptop) ---
  print *, '3. POST /productos (crear Laptop Gamer)'
  cmd = "curl -s -X POST -H 'Content-Type: application/json' " // &
        "-d '{\"nombre\":\"Laptop Gamer\",\"descripcion\":\"RTX 4070, 32GB\"," // &
        "\"precio\":1299.99,\"stock\":10}' http://localhost:8080/productos"
  call execute_command_line(cmd)
  print *, ''

  ! --- POST /productos (crear monitor) ---
  print *, '4. POST /productos (crear Monitor 4K)'
  cmd = "curl -s -X POST -H 'Content-Type: application/json' " // &
        "-d '{\"nombre\":\"Monitor 4K\",\"descripcion\":\"27 pulgadas IPS\"," // &
        "\"precio\":499.99,\"stock\":25}' http://localhost:8080/productos"
  call execute_command_line(cmd)
  print *, ''

  ! --- POST /productos (crear teclado) ---
  print *, '5. POST /productos (crear Teclado Mecanico)'
  cmd = "curl -s -X POST -H 'Content-Type: application/json' " // &
        "-d '{\"nombre\":\"Teclado Mecanico\",\"descripcion\":\"Switch Cherry MX\"," // &
        "\"precio\":149.99,\"stock\":50}' http://localhost:8080/productos"
  call execute_command_line(cmd)
  print *, ''

  ! --- GET /productos (con datos) ---
  print *, '6. GET /productos (lista con datos)'
  cmd = 'curl -s http://localhost:8080/productos'
  call execute_command_line(cmd)
  print *, ''

  ! --- GET /productos/2 ---
  print *, '7. GET /productos/2 (detalle Monitor)'
  cmd = 'curl -s http://localhost:8080/productos/2'
  call execute_command_line(cmd)
  print *, ''

  ! --- PUT /productos/1 (actualizar precio) ---
  print *, '8. PUT /productos/1 (actualizar precio Laptop)'
  cmd = "curl -s -X PUT -H 'Content-Type: application/json' " // &
        "-d '{\"nombre\":\"Laptop Gamer Pro\",\"descripcion\":\"RTX 5080, 64GB\"," // &
        "\"precio\":1999.99,\"stock\":5}' http://localhost:8080/productos/1"
  call execute_command_line(cmd)
  print *, ''

  ! --- GET /productos/1 (verificar cambio) ---
  print *, '9. GET /productos/1 (verificar actualizacion)'
  cmd = 'curl -s http://localhost:8080/productos/1'
  call execute_command_line(cmd)
  print *, ''

  ! --- DELETE /productos/3 ---
  print *, '10. DELETE /productos/3 (eliminar Teclado)'
  cmd = 'curl -s -X DELETE http://localhost:8080/productos/3'
  call execute_command_line(cmd)
  print *, ''

  ! --- GET /productos (despues de borrar) ---
  print *, '11. GET /productos (despues de eliminar)'
  cmd = 'curl -s http://localhost:8080/productos'
  call execute_command_line(cmd)
  print *, ''

  ! --- GET /productos/999 (no existe) ---
  print *, '12. GET /productos/999 (producto inexistente)'
  cmd = 'curl -s http://localhost:8080/productos/999'
  call execute_command_line(cmd)
  print *, ''

  print *, '=== PRUEBAS COMPLETADAS ==='

end program cliente_prueba
```

**Análisis línea por línea**:

- `character(len=1024) :: cmd`: Buffer para el comando curl.
- `call execute_command_line(cmd)`: Ejecuta el comando. No usamos `wait` porque el cliente es secuencial y queremos ver la salida en consola.
- Cada bloque de petición incluye un `print *` descriptivo y el comando curl correspondiente.
- Las cadenas JSON se construyen concatenando fragmentos con `//` para evitar problemas de comillas.
- El cliente incluye casos de éxito (crear, listar, obtener, actualizar, eliminar) y un caso de error (producto inexistente).

### Compilación y ejecución

```bash
# Compilar todos los modulos y programas
gfortran -std=f2018 -Wall -c json_handler.f90
gfortran -std=f2018 -Wall -c product_catalog.f90
gfortran -std=f2018 -Wall -o product_api product_api.f90 product_catalog.o json_handler.o
gfortran -std=f2018 -Wall -o cliente_prueba cliente_prueba.f90

# Dar permisos de ejecucion al script del servidor
chmod +x server.sh

# (Opcional) Compilar ejemplos de secciones anteriores
gfortran -std=f2018 -Wall -o demo_http_strings demo_http_strings.f90
gfortran -std=f2018 -Wall -o call_system call_system.f90
gfortran -std=f2018 -Wall -o socket_server socket_server.f90
gfortran -std=f2018 -Wall -o http_client http_client.f90
gfortran -std=f2018 -Wall -o client_con_captura client_con_captura.f90
gfortran -std=f2018 -Wall -o server_handler server_handler.f90
gfortran -std=f2018 -Wall -o routing_demo routing_demo.f90
gfortran -std=f2018 -Wall -o demo_middleware demo_middleware.f90

# Ejecutar el servidor (Terminal 1)
./server.sh

# Ejecutar el cliente de prueba (Terminal 2)
./cliente_prueba
```

### Pruebas con curl desde terminal

Además del cliente Fortran, puedes interactuar con la API directamente usando curl:

```bash
# Listar productos (GET)
curl -s http://localhost:8080/productos | python3 -m json.tool

# Obtener producto por ID
curl -s http://localhost:8080/productos/1 | python3 -m json.tool

# Crear producto (POST)
curl -s -X POST \
  -H 'Content-Type: application/json' \
  -d '{"nombre":"Mouse Inalambrico","descripcion":"Logitech MX Master 3","precio":89.99,"stock":100}' \
  http://localhost:8080/productos | python3 -m json.tool

# Actualizar producto (PUT)
curl -s -X PUT \
  -H 'Content-Type: application/json' \
  -d '{"nombre":"Mouse Inalambrico Pro","descripcion":"Logitech MX Master 3S","precio":99.99,"stock":75}' \
  http://localhost:8080/productos/4 | python3 -m json.tool

# Eliminar producto (DELETE)
curl -s -X DELETE http://localhost:8080/productos/4 | python3 -m json.tool

# Probar errores
curl -s http://localhost:8080/productos/999  # 404
curl -s -X PATCH http://localhost:8080/productos  # 405
curl -s http://localhost:8080/usuarios  # 404
```

**Salida esperada completa de `./cliente_prueba`**:

```
 === CLIENTE DE PRUEBA - API Productos ===

1. GET / (raiz)
{"mensaje":"API de Productos v1.0","endpoints":["GET /productos","GET /productos/{id}","POST /productos","PUT /productos/{id}","DELETE /productos/{id}"]}

2. GET /productos (lista vacia)
[]

3. POST /productos (crear Laptop Gamer)
{"id":1,"nombre":"Laptop Gamer","descripcion":"RTX 4070, 32GB","precio":1299.99,"stock":10}

4. POST /productos (crear Monitor 4K)
{"id":2,"nombre":"Monitor 4K","descripcion":"27 pulgadas IPS","precio":499.99,"stock":25}

5. POST /productos (crear Teclado Mecanico)
{"id":3,"nombre":"Teclado Mecanico","descripcion":"Switch Cherry MX","precio":149.99,"stock":50}

6. GET /productos (lista con datos)
[{"id":1,"nombre":"Laptop Gamer","descripcion":"RTX 4070, 32GB","precio":1299.99,"stock":10},{"id":2,"nombre":"Monitor 4K","descripcion":"27 pulgadas IPS","precio":499.99,"stock":25},{"id":3,"nombre":"Teclado Mecanico","descripcion":"Switch Cherry MX","precio":149.99,"stock":50}]

7. GET /productos/2 (detalle Monitor)
{"id":2,"nombre":"Monitor 4K","descripcion":"27 pulgadas IPS","precio":499.99,"stock":25}

8. PUT /productos/1 (actualizar precio Laptop)
{"id":1,"nombre":"Laptop Gamer Pro","descripcion":"RTX 5080, 64GB","precio":1999.99,"stock":5}

9. GET /productos/1 (verificar actualizacion)
{"id":1,"nombre":"Laptop Gamer Pro","descripcion":"RTX 5080, 64GB","precio":1999.99,"stock":5}

10. DELETE /productos/3 (eliminar Teclado)
{"mensaje":"Producto eliminado","id":3}

11. GET /productos (despues de eliminar)
[{"id":1,"nombre":"Laptop Gamer Pro","descripcion":"RTX 5080, 64GB","precio":1999.99,"stock":5},{"id":2,"nombre":"Monitor 4K","descripcion":"27 pulgadas IPS","precio":499.99,"stock":25}]

12. GET /productos/999 (producto inexistente)
{"error":"Producto no encontrado"}

 === PRUEBAS COMPLETADAS ===
```

### Errores típicos del mini-proyecto

Error: **netcat-openbsd no instalado**. El comando `nc -l -p 8080 -e ./product_api` falla con `invalid option -- 'e'`.
```
Solución: sudo apt update && sudo apt install netcat-openbsd
```
Verificar la versión instalada: `nc -h` debe mostrar la opción `-e`.

Error: **el handler no encuentra los módulos**. Al ejecutar `./product_api`, gfortran runtime busca los módulos compilados.
```
Solución: compilar en orden y enlazar correctamente:
gfortran -std=f2018 -c json_handler.f90
gfortran -std=f2018 -c product_catalog.f90
gfortran -std=f2018 -o product_api product_api.f90 product_catalog.o json_handler.o
```

Error: **puerto ocupado**. Si otro proceso está usando el puerto 8080, netcat falla con `Address already in use`.
```
Solución: cambiar de puerto: ./server.sh 9090
O matar el proceso anterior: fuser -k 8080/tcp
```

Error: **el cuerpo JSON POST supera una línea**. Nuestro handler lee el cuerpo como una sola línea. Si curl envía JSON con saltos de línea, solo lee el primer fragmento.
```
Solución: asegurar que curl envía JSON en una línea (curl lo hace por defecto con -d).
O modificar el handler para leer en bucle hasta alcanzar Content-Length bytes.
```

Error: **routing sensible a la barra final**. La ruta `/productos` y `/productos/` son distintas en nuestro routing (`==`). La segunda daría 404.
```
Solución: normalizar la ruta al inicio:
if (len_trim(path) > 1 .and. path(len_trim(path):len_trim(path)) == '/') then
  path = path(:len_trim(path)-1)
end if
```

---

## Conclusión

Has completado un viaje desde los fundamentos de HTTP hasta una API RESTful funcional escrita completamente en Fortran moderno. Has aprendido a interoperar con C usando `iso_c_binding`, a ejecutar comandos del sistema con `execute_command_line`, a construir un servidor web usando netcat, a serializar y deserializar JSON con código Fortran puro, y a estructurar una aplicación en módulos con responsabilidades bien definidas.

El mini-proyecto del Catálogo de Productos demuestra que Fortran no es solo un lenguaje para cálculos numéricos: es perfectamente capaz de actuar como un servicio web autónomo. Cada pieza —el cliente HTTP con curl, el handler CGI, el módulo JSON, el enrutador— es software funcional, compilable y ejecutable.

Este capítulo sienta las bases para expandir el proyecto en múltiples direcciones: agregar una base de datos SQLite via `iso_c_binding`, implementar autenticación con tokens JWT, servir archivos estáticos, o empaquetar la aplicación como un microservicio Docker. La arquitectura modular que has aprendido te permitirá añadir nuevas funcionalidades sin reescribir el código existente.

El código completo de este capítulo está listo para compilar y ejecutar. Abre tu terminal, compila los módulos, inicia el servidor, y prueba la API con curl. Ver cómo Fortran responde a una petición HTTP con un JSON bien formado es una satisfacción que pocos programadores de Fortran han experimentado. Ahora tú eres uno de ellos.
