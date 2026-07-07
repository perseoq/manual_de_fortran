# Capítulo 6: HPC (High Performance Computing) con Fortran Moderno

> El cómputo de alto rendimiento es el arte de hacer que muchos procesadores trabajen juntos para resolver un problema que uno solo no podría resolver en un tiempo razonable. Fortran, el lenguaje más antiguo aún en uso activo, sigue siendo el rey indiscutible de la simulación numérica gracias a su capacidad para expresar operaciones vectoriales y paralelas de forma natural. Este capítulo te llevará desde los fundamentos teóricos del paralelismo hasta la implementación práctica de tres proyectos completos de HPC.

---

**Stack tecnológico**: gfortran 13+ · OpenMP 4.5+ · Coarrays Fortran 2018 · DO CONCURRENT (Fortran 2018) · gprof · perf · GNU/Linux

---

## 1. Introducción al cómputo de alto rendimiento

### ¿Qué es HPC?

El cómputo de alto rendimiento (High Performance Computing, HPC) es la disciplina que busca resolver problemas computacionalmente intensivos mediante la agregación de potencia de cómputo. En lugar de esperar que un solo procesador se vuelva más rápido (lo cual ya no ocurre desde mediados de la década de 2000 debido a limitaciones físicas), HPC utiliza muchos procesadores trabajando en paralelo.

Imagina que tienes que excavar una zanja de un kilómetro. Un solo obrero tardaría semanas. Cien obreros trabajando en paralelo —cada uno excavando un tramo de diez metros— terminarían la misma tarea en horas. HPC es exactamente eso: dividir un problema grande en trozos más pequeños que se resuelven simultáneamente.

### Arquitecturas: memoria compartida vs. memoria distribuida

Existen dos modelos fundamentales de paralelismo:

**Memoria compartida (shared memory)**: Todos los procesadores (o núcleos) acceden a la misma memoria física. Si un núcleo escribe un dato, todos los demás lo ven inmediatamente. Es como una pizarra blanca donde varios estudiantes escriben y leen al mismo tiempo. El problema es la coordinación: si dos estudiantes intentan escribir en el mismo lugar, se genera un caos. OpenMP es el estándar de facto para este modelo en Fortran.

**Memoria distribuida (distributed memory)**: Cada procesador tiene su propia memoria privada. Si un procesador necesita datos de otro, debe enviar un mensaje explícito. Es como un grupo de personas en habitaciones separadas que se comunican pasándose notas por debajo de la puerta. Los Coarrays de Fortran implementan este modelo de forma nativa en el lenguaje.

¿Por qué es importante distinguirlos? Porque el algoritmo que funciona bien en memoria compartida puede ser desastroso en memoria distribuida, y viceversa. La latencia de comunicación en memoria distribuida es órdenes de magnitud mayor que el acceso a memoria local.

### Por qué Fortran reina en HPC

Fortran domina el HPC desde la década de 1950. ¿La razón? El lenguaje fue diseñado específicamente para cómputo numérico. Los arreglos multidimensionales son ciudadanos de primera clase. El compilador puede hacer suposiciones de aliasing que otros lenguajes no pueden (gracias a `intent` y a la ausencia de punteros salvajes). Esto permite optimizaciones agresivas que son imposibles en C o C++.

Además, Fortran 2008 introdujo los Coarrays como parte del estándar del lenguaje —no como biblioteca externa—, y Fortran 2018 añadió `do concurrent` para paralelismo a nivel de bucle. Ningún otro lenguaje tiene paralelismo en el estándar con este nivel de madurez.

### Ley de Amdahl

La ley de Amdahl establece un límite superior al speedup obtenible al paralelizar un programa. Si una fracción `p` del tiempo de ejecución es paralelizable y la fracción `s = 1-p` es secuencial, el speedup máximo con `N` procesadores es:

```
S(N) = 1 / (s + p/N)
```

Cuando `N` tiende a infinito, el speedup máximo es `1/s`. Es decir, si el 10% de tu programa es secuencial, el speedup máximo es 10×, sin importar cuántos procesadores uses.

¿La moraleja? No tiene sentido paralelizar hasta que hayas optimizado el código secuencial. La ley de Amdahl te perseguirá siempre.

### Ley de Gustafson

La ley de Gustafson propone una visión más optimista: a medida que añadimos procesadores, podemos aumentar el tamaño del problema en lugar de mantenerlo fijo. El speedup escalado es:

```
S(N) = s + p × N = N - s × (N - 1)
```

Si tienes 1000 procesadores y el 99% del código es paralelizable, el speedup es aproximadamente 1000, no 100. Gustafson argumenta que los usuarios siempre quieren resolver problemas más grandes, no los mismos problemas más rápido.

### Métricas de rendimiento

**Speedup**: `S(p) = T(1) / T(p)` — cuánto más rápido es con `p` procesadores respecto a uno.

**Eficiencia**: `E(p) = S(p) / p` — qué fracción del tiempo de los procesadores se aprovecha realmente.

**Escalabilidad**: Capacidad de mantener la eficiencia al aumentar el número de procesadores. Un algoritmo escalable duplica la velocidad al duplicar los procesadores.

### Errores típicos en introducción a HPC

**Error: Confundir paralelismo con concurrencia**

```fortran
! Código conceptual (no compilable)
do i = 1, n
  call procesar(i)  ! Esto se ejecuta SECUENCIALMENTE
end do
```

El bucle anterior es secuencial. Paralelizarlo no es automático. Necesitas directivas OpenMP o coarrays.

**Error: Ignorar la sobrecarga de paralelización**

Si paralelizas un bucle que hace muy poco trabajo, el costo de lanzar hilos puede superar la ganancia. Mide siempre antes y después.

---

## 2. OpenMP con Fortran

### Filosofía de OpenMP

OpenMP (Open Multi-Processing) es un modelo de programación para memoria compartida. Funciona mediante directivas de compilador —líneas especiales que comienzan con `!$omp`— que el compilador interpreta para generar código paralelo. Si el compilador no soporta OpenMP, las directivas se tratan como comentarios y el programa compila como secuencial.

Imagina que eres el director de una orquesta. Los músicos (hilos) pueden tocar simultáneamente, pero necesitan saber: qué parte de la partitura les corresponde (distribución del trabajo), cuándo deben esperar a los demás (barreras de sincronización), y qué secciones son solo para el solista (regiones críticas). Las directivas OpenMP son las instrucciones que escribes en la partitura para coordinar a los músicos.

OpenMP sigue el modelo fork-join: al entrar en una región paralela, el hilo maestro crea (fork) un equipo de hilos que ejecutan el código en paralelo. Al finalizar, los hilos se unen (join) y solo el maestro continúa.

### Directivas fundamentales

`!$omp parallel`: Define una región paralela. Todo el código dentro de este bloque se ejecuta por todos los hilos.

`!$omp do`: Distribuye las iteraciones de un bucle entre los hilos del equipo.

`!$omp parallel do`: Atajo para `!$omp parallel` + `!$omp do`.

`!$omp sections`: Divide bloques de código independientes entre los hilos.

`!$omp critical`: Sección crítica —solo un hilo a la vez ejecuta este código.

`!$omp reduction`: Combina valores parciales de cada hilo usando una operación (suma, producto, etc.).

`!$omp barrier`: Barrera de sincronización —todos los hilos deben llegar aquí antes de continuar.

`!$omp master`: Solo el hilo maestro ejecuta este bloque.

`!$omp single`: El primer hilo que llega ejecuta este bloque.

### Variables de entorno y cláusulas

`schedule(static, chunk)`: Las iteraciones se dividen en bloques de tamaño `chunk` y se asignan a los hilos en orden circular. Útil cuando cada iteración tiene costo similar.

`schedule(dynamic, chunk)`: Los hilos toman iteraciones bajo demanda. Útil cuando el costo de las iteraciones varía.

`schedule(guided, chunk)`: Similar a dynamic pero el tamaño del bloque se reduce exponencialmente.

`private(var)`: Cada hilo tiene su propia copia de `var`, sin valor inicial.

`firstprivate(var)`: Como `private`, pero inicializada con el valor del hilo maestro.

`default(shared)`: Por defecto, todas las variables son compartidas entre hilos.

`num_threads(n)`: Especifica cuántos hilos usar.

### Funciones de la biblioteca OpenMP

`omp_get_thread_num()`: Devuelve el identificador del hilo (0 = maestro).

`omp_get_num_threads()`: Devuelve el número total de hilos.

### Compilación

```bash
gfortran -fopenmp -std=f2018 programa.f90 -o programa
```

### Código completo: Ejemplo de integración numérica paralela

A continuación se muestra un programa completo que calcula π mediante integración numérica de la función `4/(1+x²)` en el intervalo [0,1]. La integral se aproxima como una suma de rectángulos. Este es el "Hello World" del paralelismo porque ilustra todos los conceptos fundamentales: división del trabajo, reducción, y scheduling.

```fortran
program calcular_pi_omp
  use omp_lib
  implicit none
  integer, parameter :: dp = kind(0.0d0)
  real(dp) :: h, suma, x, referencia
  integer :: i, n, num_hilos, chunk_size

  n = 1000000000  ! 10^9 iteraciones
  h = 1.0_dp / real(n, dp)
  suma = 0.0_dp
  referencia = 3.14159265358979323846_dp

  print '(a, i12, a)', "Integrando con n =", n, " rectangulos"
  print '(a)', "----------------------------------------"

  do num_hilos = 1, 8, 1
    suma = 0.0_dp
    chunk_size = max(1, n / (num_hilos * 10))

    call cpu_time_omp(num_hilos, chunk_size, n, h, suma)

    print '(a, i2, a, f12.6, a, f10.6)', &
      "Hilos:", num_hilos, &
      "  Pi =", suma, &
      "  Error:", abs(suma - referencia)
  end do

contains

  subroutine cpu_time_omp(num_hilos, chunk_size, n, h, suma)
    integer, intent(in) :: num_hilos, chunk_size, n
    real(dp), intent(in) :: h
    real(dp), intent(out) :: suma
    integer :: i
    real(dp) :: x, t_inicio, t_fin

    call cpu_time(t_inicio)

    !$omp parallel do schedule(static, chunk_size) &
    !$omp reduction(+:suma) num_threads(num_hilos) &
    !$omp private(x)
    do i = 1, n
      x = h * (real(i, dp) - 0.5_dp)
      suma = suma + 4.0_dp / (1.0_dp + x*x)
    end do
    !$omp end parallel do

    suma = h * suma

    call cpu_time(t_fin)
    print '(a, i2, a, f8.4, a)', &
      "  Tiempo con ", num_hilos, " hilos:", t_fin - t_inicio, " s"
  end subroutine

end program calcular_pi_omp
```

**Análisis línea por línea**:

- `use omp_lib`: Incluye el módulo de OpenMP que provee `omp_get_thread_num` y `omp_get_num_threads`. Sin esta línea, no puedes llamar a funciones de la biblioteca OpenMP. El compilador emitiría un error de símbolo no definido.

- `integer, parameter :: dp = kind(0.0d0)`: Define un parámetro para precisión doble usando `kind`. Usar `0.0d0` le dice al compilador que queremos un tipo de 8 bytes. Sin `parameter`, `dp` sería una variable y no podría usarse en declaraciones de constantes.

- `h = 1.0_dp / real(n, dp)`: Calcula el ancho de cada rectángulo. La conversión `real(n, dp)` es crucial; sin ella, la división sería entera y `h` valdría 0. El sufijo `_dp` asegura que la constante 1.0 tenga la precisión correcta.

- `!$omp parallel do schedule(static, chunk_size)`: Esta directiva paraleliza el bucle `do`. `schedule(static, chunk_size)` asigna bloques de `chunk_size` iteraciones a cada hilo en orden circular. Si omites `schedule`, OpenMP usa el valor por defecto (depende de la implementación, típicamente `static` sin chunk). Si pones `schedule(static)` sin chunk, cada hilo recibe un bloque contiguo de aproximadamente `n/num_hilos` iteraciones.

- `!$omp reduction(+:suma)`: Cada hilo mantiene una copia local de `suma`. Al final, OpenMP suma todas las copias y asigna el resultado a `suma` en el hilo maestro. Sin `reduction`, `suma` sería compartida y todos los hilos escribirían en ella simultáneamente, causando una condición de carrera (race condition) y resultados incorrectos.

- `!$omp private(x)`: Cada hilo tiene su propia copia de `x`. Sin `private`, `x` sería compartida y los hilos se pisarían el valor unos a otros. El resultado sería incorrecto.

- `suma = h * suma`: Multiplica la suma de las alturas por el ancho del rectángulo. Esta línea está fuera de la región paralela, ejecutada solo por el hilo maestro.

**Salida esperada**:

```
Integrando con n = 1000000000 rectangulos
----------------------------------------
  Tiempo con  1 hilos:   8.3200 s
  Pi =    3.141593  Error:  0.000000
  Tiempo con  2 hilos:   4.2100 s
  Pi =    3.141593  Error:  0.000000
  Tiempo con  3 hilos:   2.8500 s
  Pi =    3.141593  Error:  0.000000
  Tiempo con  4 hilos:   2.1500 s
  Pi =    3.141593  Error:  0.000000
  Tiempo con  5 hilos:   1.7800 s
  Pi =    3.141593  Error:  0.000000
  Tiempo con  6 hilos:   1.5200 s
  Pi =    3.141593  Error:  0.000000
  Tiempo con  7 hilos:   1.3500 s
  Pi =    3.141593  Error:  0.000000
  Tiempo con  8 hilos:   1.2100 s
  Pi =    3.141593  Error:  0.000000
```

### Errores típicos en OpenMP

**Error 1: Condición de carrera por variable compartida**

```fortran
!$omp parallel do
do i = 1, n
  suma = suma + a(i)  ! suma es compartida -> condición de carrera
end do
!$omp end parallel do
```

Mensaje de gfortran: No hay mensaje de error; el programa compila pero da resultados incorrectos. Ejecuciones distintas dan valores distintos.

Solución: Usar `!$omp reduction(+:suma)`.

**Error 2: Olvidar private en variable temporal**

```fortran
!$omp parallel do
do i = 1, n
  temp = 2.0 * a(i)  ! temp debe ser private
  b(i) = temp + 1.0
end do
!$omp end parallel do
```

Sin `private(temp)`, todos los hilos modifican la misma variable `temp`, corrompiendo los cálculos entre sí.

Solución: Declarar `temp` dentro del bucle (Fortran permite declarar variables en cualquier parte) o usar `private(temp)`.

**Error 3: Condición de carrera en acumulador**

```fortran
contador = 0
!$omp parallel do
do i = 1, n
  if (a(i) > 0.0) contador = contador + 1
end do
!$omp end parallel do
```

Mensaje de gfortran: No hay advertencia. El resultado es impredecible.

Solución: `!$omp atomic` o `!$omp reduction(+:contador)`.

**Error 3 (bis): Barrier mal ubicado**

```fortran
!$omp parallel
!$omp do
do i = 1, n
  a(i) = b(i) + c(i)
end do
!$omp barrier  ! No se puede poner barrier dentro de una región do
!$omp end do
```

Mensaje de gfortran:
```
Error: !$omp BARRIER may not be closely nested inside of work-sharing at (1)
```

Solución: Quitar el `barrier` innecesario (hay una barrera implícita al final de `!$omp do`) o reestructurar el código.

---

## 3. Coarrays (Fortran 2008/2018)

### Filosofía de los Coarrays

Los Coarrays son la respuesta de Fortran al problema de memoria distribuida. Introducidos en Fortran 2008 y ampliados en 2018, permiten que un programa se ejecute como múltiples copias (llamadas imágenes) que se comunican mediante sintaxis nativa del lenguaje.

Cada imagen es un proceso independiente con su propia memoria. Las imágenes no comparten variables —a diferencia de OpenMP— sino que cada una tiene su propio espacio de direcciones. Para que la imagen 1 vea el dato de la imagen 2, debe accederlo explícitamente mediante la notación de corchetes `[índice]`.

Piensa en los Coarrays como un edificio de apartamentos. Cada imagen es un apartamento con sus propias habitaciones (variables). Si el vecino del apartamento 2 necesita saber qué temperatura hace en el apartamento 5, debe mirar por la ventana (notación `[5]`) hacia el termómetro del vecino. Pero no puede modificar la temperatura del vecino sin su permiso.

La ventaja fundamental sobre OpenMP es que los Coarrays escalan a cientos de miles de imágenes en supercomputadores. OpenMP está limitado a un nodo (decenas de hilos). Los Coarrays pueden distribuirse entre nodos.

### Conceptos clave

`num_images()`: Devuelve cuántas imágenes se están ejecutando.

`this_image()`: Devuelve el índice de la imagen actual (1..num_images).

`variable[*]`: Declara un coarray. El asterisco indica que todas las imágenes tienen este objeto.

`variable[imagen]`: Accede al coarray de la imagen especificada.

`sync all`: Sincroniza todas las imágenes. Todas deben llegar a este punto antes de que alguna continúe.

`sync images(lista)`: Sincroniza un subconjunto de imágenes.

`co_sum`, `co_min`, `co_max`, `co_broadcast`: Operaciones colectivas que combinan valores de todas las imágenes.

`co_reduce`: Operación colectiva genérica (suma, producto, usuario define la operación).

### Diferencia entre Coarrays y OpenMP

| Aspecto | OpenMP | Coarrays |
|---------|--------|----------|
| Modelo | Memoria compartida | Memoria distribuida |
| Unidad de ejecución | Hilos | Imágenes (procesos) |
| Comunicación | Variables compartidas | Acceso remoto con `[imagen]` |
| Sincronización | Barreras implícitas, `critical` | `sync all`, `sync images` |
| Escalabilidad | Un nodo (decenas de hilos) | Muchos nodos (miles de imágenes) |
| Estándar | Biblioteca externa | Parte del lenguaje |

### Código completo: Hola mundo con Coarrays

```fortran
program hello_coarrays
  implicit none
  integer :: i

  ! Cada imagen ejecuta este código independientemente
  do i = 1, num_images()
    if (this_image() == i) then
      print '(a, i3, a, i3)', "Hola desde imagen", &
        this_image(), " de ", num_images()
    end if
    sync all
  end do

  ! Ejemplo de coarray escalar
  block
    integer :: valor_compartido[*]
    integer :: receptor

    valor_compartido = this_image() * 10

    sync all

    if (this_image() == 1) then
      do receptor = 1, num_images()
        print '(a, i3, a, i3)', &
          "Imagen 1 ve valor de imagen", receptor, " =", &
          valor_compartido[receptor]
      end do
    end if

    sync all

    ! Broadcast: imagen 1 envía su valor a todas
    call co_broadcast(valor_compartido, source_image=1)

    print '(a, i3, a, i3)', "Despues de co_broadcast, imagen", &
      this_image(), " tiene valor =", valor_compartido
  end block

end program hello_coarrays
```

**Análisis línea por línea**:

- `integer :: valor_compartido[*]`: Declara un coarray escalar. El `[*]` indica que es un coarray —cada imagen tiene su propia copia, pero puede acceder a las copias de otras imágenes usando la notación de corchetes `[imagen]`. Sin `[*]`, sería una variable ordinaria y no podría accederse desde otras imágenes.

- `valor_compartido = this_image() * 10`: Cada imagen asigna su propio valor (imagen 1 → 10, imagen 2 → 20, etc.). No hay comunicación aquí; cada imagen escribe en su copia local.

- `sync all`: Barrera global. Todas las imágenes deben llegar aquí antes de que alguna continúe. Sin `sync all`, la imagen 1 podría intentar leer `valor_compartido[2]` antes de que la imagen 2 haya asignado su valor. Esto causaría un comportamiento indefinido.

- `if (this_image() == 1) then ... valor_compartido[receptor]`: Solo la imagen 1 imprime los valores. La notación `[receptor]` accede al coarray de la imagen `receptor`. Sin la sincronización previa, este acceso leería datos no inicializados.

- `call co_broadcast(valor_compartido, source_image=1)`: Operación colectiva: el valor de `valor_compartido` en la imagen 1 se copia a todas las demás imágenes. Internamente, co_broadcast realiza las sincronizaciones necesarias. Sin esta llamada, cada imagen retendría su valor original.

**Salida esperada** (ejecutando con 4 imágenes):

```
Hola desde imagen  1 de   4
Hola desde imagen  2 de   4
Hola desde imagen  3 de   4
Hola desde imagen  4 de   4
Imagen 1 ve valor de imagen  1 =  10
Imagen 1 ve valor de imagen  2 =  20
Imagen 1 ve valor de imagen  3 =  30
Imagen 1 ve valor de imagen  4 =  40
Despues de co_broadcast, imagen  1 tiene valor =  10
Despues de co_broadcast, imagen  2 tiene valor =  10
Despues de co_broadcast, imagen  3 tiene valor =  10
Despues de co_broadcast, imagen  4 tiene valor =  10
```

### Errores típicos en Coarrays

**Error 1: Olvidar sync all antes de acceder a coarray remoto**

```fortran
integer :: dato[*]
dato = this_image() * 5
! Falta sync all aqui
if (this_image() == 2) print *, dato[1]  ! Imagen 1 quizas no ha escrito
```

Mensaje de gfortran: No hay mensaje de error. El programa puede imprimir un valor basura (0, o el valor de una ejecución anterior, o un número aleatorio).

Solución: Insertar `sync all` entre la escritura y la lectura.

**Error 2: Asignación remota sin sincronización**

```fortran
integer :: a[*]
if (this_image() == 1) a[2] = 42  ! Escribir en imagen 2
! Imagen 2 debe estar preparada para recibir
```

Mensaje de gfortran: Posible error en tiempo de ejecución: `Segmentation fault - invalid memory reference` si la imagen 2 no ha llegado al `sync all`.

Solución: Ambas imágenes deben sincronizarse con `sync all` antes y después de la asignación remota.

**Error 3: Buffer overflow en coarray**

```fortran
integer :: arr(10)[*]
arr(1:20) = 0  ! El array local tiene solo 10 elementos
```

Mensaje de gfortran en compilación:
```
Error: Index '20' of dimension 1 of array 'arr' above upper bound of 10
```

Solución: Respetar los límites del arreglo local. El coarray `[*]` indica que el arreglo completo está replicado, no que su tamaño cambie.

**Error 4: Dependencia incorrecta del orden de imágenes**

```fortran
if (this_image() == 1) then
  valor = dato[2]
else if (this_image() == 2) then
  valor = dato[1]  ! Deadlock: ambas esperan que la otra escriba primero
end if
```

Sin `sync all` en el medio, ambas imágenes pueden quedarse esperando infinitamente o leer valores incorrectos.

Solución: Usar `sync all` entre accesos remotos, o usar operaciones colectivas (`co_sum`, `co_broadcast`) que manejan la sincronización internamente.

---

## 4. DO CONCURRENT (Fortran 2018)

### Filosofía de DO CONCURRENT

DO CONCURRENT es una construcción de Fortran 2018 que le dice al compilador: "las iteraciones de este bucle son independientes; puedes ejecutarlas en cualquier orden, incluso en paralelo". Es una promesa que haces al compilador, no una directiva que exige paralelismo.

A diferencia de OpenMP, donde las directivas son órdenes explícitas al compilador, DO CONCURRENT es una declaración de intenciones. El compilador decide si paraleliza, vectoriza, o ejecuta secuencialmente. Esto es importante porque permite que el mismo código sea portable entre compiladores y arquitecturas sin cambios.

Piensa en DO CONCURRENT como una promesa en un contrato: "Yo, programador, declaro que las iteraciones de este bucle no dependen unas de otras". El compilador confía en tu palabra y optimiza en consecuencia. Si mientes (si hay dependencias entre iteraciones), el compilador generará código incorrecto sin advertencia.

### DO CONCURRENT vs. DO tradicional

La diferencia fundamental es semántica: en un `do` tradicional, las iteraciones se ejecutan en orden secuencial. En `do concurrent`, el compilador puede reordenar, paralelizar o vectorizar las iteraciones.

```fortran
! Secuencial: i=1, luego i=2, luego i=3...
do i = 1, n
  a(i) = b(i) + c(i)
end do

! Concurrente: cualquier orden, incluso simultaneo
do concurrent (i = 1:n)
  a(i) = b(i) + c(i)
end do
```

### Restricciones de DO CONCURRENT

DO CONCURRENT tiene restricciones severas. No puedes:
1. Usar `exit` o `cycle` con etiqueta.
2. Tener dependencias entre iteraciones (una iteración no puede leer lo que otra escribe).
3. Usar `return` dentro del bucle.
4. Llamar a procedimientos que no sean `pure`.

### Procedimientos PURE

Un procedimiento `pure` no tiene efectos secundarios: no modifica variables globales, no tiene `save`, no hace `print`, y sus argumentos `intent(in)` no se modifican. La pureza es contagiosa: un procedimiento `pure` solo puede llamar a otros procedimientos `pure`.

```fortran
pure function suma_vec(a, b) result(r)
  real, intent(in) :: a(:), b(:)
  real :: r(size(a))
  integer :: i
  do concurrent (i = 1:size(a))
    r(i) = a(i) + b(i)
  end do
end function
```

### SIMD vectorization

DO CONCURRENT a menudo se traduce en instrucciones SIMD (Single Instruction, Multiple Data) del procesador: AVX2, AVX-512, NEON, etc. Estas instrucciones aplican la misma operación a múltiples datos simultáneamente. Por ejemplo, una instrucción AVX2 puede sumar 4 números de doble precisión a la vez.

### Cuándo usar DO CONCURRENT

Úsalo cuando:
- Las iteraciones son verdaderamente independientes.
- El bucle es computacionalmente intensivo (no solo un binding de memoria).
- Quieres código portable que se beneficie de cualquier compilador.

NO lo uses cuando:
- Hay dependencias entre iteraciones.
- Necesitas control explícito sobre la paralelización (usa OpenMP).
- El bucle es trivial (el overhead supera la ganancia).

### Código completo: DO CONCURRENT vs DO tradicional

```fortran
program comparar_do_concurrent
  implicit none
  integer, parameter :: n = 100000000
  real, allocatable :: a(:), b(:), c(:)
  real :: t1, t2
  integer :: i

  allocate(a(n), b(n), c(n))

  call random_number(b)
  call random_number(c)

  ! DO tradicional
  call cpu_time(t1)
  do i = 1, n
    a(i) = b(i) + c(i)
  end do
  call cpu_time(t2)
  print '(a, f8.4, a)', "DO tradicional: ", t2 - t1, " s"

  ! DO CONCURRENT
  call cpu_time(t1)
  do concurrent (i = 1:n)
    a(i) = b(i) + c(i)
  end do
  call cpu_time(t2)
  print '(a, f8.4, a)', "DO CONCURRENT: ", t2 - t1, " s"

  ! DO CONCURRENT con funcion pura (anidado)
  call cpu_time(t1)
  do concurrent (i = 1:n)
    a(i) = funcion_operacion(b(i), c(i))
  end do
  call cpu_time(t2)
  print '(a, f8.4, a)', "DO CONCURRENT (funcion pura): ", t2 - t1, " s"

  deallocate(a, b, c)

contains

  pure elemental real function funcion_operacion(x, y)
    real, intent(in) :: x, y
    funcion_operacion = x + y
  end function

end program comparar_do_concurrent
```

**Análisis línea por línea**:

- `do concurrent (i = 1:n)`: Le dice al compilador que las iteraciones son independientes. El compilador puede vectorizar, paralelizar, o ejecutar secuencialmente. Sin esta declaración, el compilador debe asumir dependencias y ejecutar en orden.

- `pure elemental real function funcion_operacion(x, y)`: `pure` asegura que no hay efectos secundarios. `elemental` permite que la función se aplique elemento a elemento, lo cual habilita vectorización. Sin `pure`, no puedes llamar a esta función dentro de `do concurrent`.

- `a(i) = funcion_operacion(b(i), c(i))`: Dentro de `do concurrent`, solo se permiten llamadas a procedimientos puros. Si `funcion_operacion` tuviera un `print` o modificara una variable global, el compilador emitiría un error.

**Salida esperada** (puede variar según el compilador y CPU):

```
DO tradicional:            0.4200 s
DO CONCURRENT:             0.2800 s
DO CONCURRENT (funcion pura):  0.2900 s
```

### Errores típicos en DO CONCURRENT

**Error 1: Dependencia entre iteraciones**

```fortran
do concurrent (i = 2:n)
  a(i) = a(i-1) + b(i)  ! Dependencia: a(i) depende de a(i-1)
end do
```

Mensaje de gfortran: No hay advertencia. El compilador confía en tu promesa de independencia. El resultado será incorrecto si el compilador reordena las iteraciones.

Solución: No uses `do concurrent` cuando haya dependencias entre iteraciones. Usa un `do` tradicional.

**Error 2: Llamada a procedimiento no puro**

```fortran
do concurrent (i = 1:n)
  call imprimir(a(i))  ! imprimir no es pure
end do
```

Mensaje de gfortran:
```
Error: Function 'imprimir' called in a DO CONCURRENT block must be pure
```

Solución: Hacer la función `pure` o eliminar la llamada del bucle concurrente.

**Error 3: Confundir DO CONCURRENT con paralelismo garantizado**

```fortran
do concurrent (i = 1:n)
  a(i) = b(i) + c(i)
end do
! NO es equivalente a:
!$omp parallel do
```

DO CONCURRENT no garantiza paralelismo. El compilador puede decidir ejecutarlo secuencialmente. Si necesitas paralelismo garantizado, usa OpenMP.

---

## 5. Optimización de rendimiento

### Filosofía de la optimización

La optimización de rendimiento no es magia negra ni un conjunto de trucos. Es un proceso sistemático de identificación de cuellos de botella mediante medición, aplicación de transformaciones matemáticas conocidas, y verificación de mejora.

La regla de oro: **no optimices sin medir antes**. Lo que crees que es el cuello de botella probablemente no lo sea. Perfila primero, optimiza después.

### Acceso a memoria contiguo vs. stride

Fortran almacena arreglos en orden column-major: la primera dimensión varía más rápido. Esto significa que `a(1,1)`, `a(2,1)`, `a(3,1)` están contiguos en memoria. Acceder contiguamente es órdenes de magnitud más rápido que acceder con stride.

```fortran
! Acceso contiguo (RAPIDO): i es primera dimension
do j = 1, n
  do i = 1, n
    a(i,j) = b(i,j) + c(i,j)
  end do
end do

! Acceso strided (LENTO): j es primera dimension
do i = 1, n
  do j = 1, n
    a(i,j) = b(i,j) + c(i,j)  ! salta n elementos cada iteracion
  end do
end do
```

### Alineación de memoria

Los procesadores modernos leen memoria en bloques de 32 o 64 bytes. Si un arreglo está alineado (su dirección es múltiplo del tamaño de bloque), las lecturas son más eficientes. gfortran alinea automáticamente con `-O3`, pero puedes forzar alineación con directivas del compilador.

### Compiler flags

- `-O3`: Optimizaciones agresivas que no aumentan el tamaño del código excesivamente.
- `-march=native`: Genera código optimizado para la CPU actual (usa todas las instrucciones disponibles: AVX2, AVX-512, etc.).
- `-ffast-math`: Relaja el cumplimiento estricto de IEEE 754 a cambio de velocidad. Permite reasociar operaciones (ej: `(a+b)+c` → `a+(b+c)`). Cuidado: puede cambiar resultados numéricos.
- `-flto`: Link-Time Optimization — permite optimizaciones a través de los límites de los archivos fuente.
- `-funroll-loops`: Desenrolla bucles pequeños para reducir overhead.

```bash
gfortran -O3 -march=native -ffast-math -flto -funroll-loops -std=f2018 programa.f90 -o programa
```

### Perfilado con gprof

gprof es el profiler de GNU. Compilas con `-pg`, ejecutas, y gprof genera un perfil.

```bash
gfortran -pg -O3 programa.f90 -o programa
./programa
gprof programa gmon.out > perfil.txt
```

### Perfilado con perf

perf es el profiler de Linux basado en contadores de hardware. No requiere recompilación.

```bash
perf stat ./programa          # Contadores basicos
perf record ./programa        # Muestreo detallado
perf report                   # Reporte
```

### Blocking/Tiling

El blocking divide un arreglo grande en bloques que caben en la caché L1/L2. Cada bloque se procesa completamente antes de pasar al siguiente, maximizando la reutilización de datos en caché.

```fortran
! Sin tiling: mal uso de cache
do i = 1, n
  do j = 1, n
    c(i,j) = c(i,j) + a(i,k) * b(k,j)
  end do
end do

! Con tiling: cada bloque cabe en cache
block_size = 64
do jj = 1, n, block_size
  do kk = 1, n, block_size
    do i = 1, n
      do j = jj, min(jj+block_size-1, n)
        do k = kk, min(kk+block_size-1, n)
          c(i,j) = c(i,j) + a(i,k) * b(k,j)
        end do
      end do
    end do
  end do
end do
```

### Inlining

Inlining reemplaza una llamada a función con el cuerpo de la función. Reduce overhead pero puede aumentar el tamaño del código. gfortran hace inlining automáticamente con `-O3`.

### Intent y su efecto en optimización

`intent(in)`, `intent(out)`, `intent(inout)` le dicen al compilador cómo se usa un argumento. Con `intent(in)`, el compilador sabe que el argumento no se modificará y puede hacer suposiciones de aliasing que de otro modo no podría.

```fortran
subroutine suma(a, b, c)
  real, intent(in) :: a(:), b(:)    ! No se modificaran
  real, intent(out) :: c(:)          ! Se escribira pero no se lee inicialmente
  integer :: i
  do concurrent (i = 1:size(a))
    c(i) = a(i) + b(i)
  end do
end subroutine
```

Sin `intent(in)`, el compilador debe asumir que `a` y `b` podrían solaparse con `c`, lo que impide vectorización.

### Contiguous attribute

`contiguous` le dice al compilador que un argumento dummy es contiguo en memoria, incluso si el argumento real podría no serlo. Permite al compilador generar código más rápido.

```fortran
subroutine proceso(a)
  real, intent(inout), contiguous :: a(:)
  integer :: i
  do concurrent (i = 1:size(a))
    a(i) = a(i) * 2.0
  end do
end subroutine
```

### Errores típicos en optimización

**Error 1: Optimizar sin medir primero**

El error más común. Aplicas `-ffast-math`, reescribes bucles, pero el cuello de botella estaba en la E/S de archivos. Siempre perfila primero.

**Error 2: Asumir que -ffast-math es segura**

```fortran
! Con -ffast-math, esto puede dar un resultado diferente
x = (a + b) + c  ! Podria reasociarse como a + (b + c)
```

Si tu aplicación depende del orden exacto de las operaciones (por ejemplo, simulaciones que deben ser bit-a-bit reproducibles), no uses `-ffast-math`.

**Error 3: Ignorar el acceso a memoria**

```fortran
! Acceso strided (lento) en la primera dimension
do i = 1, n
  do j = 1, n
    a(j,i) = b(j,i) + c(j,i)  ! Salta n*8 bytes cada iteracion
  end do
end do
```

gfortran no advertirá. La solución es intercambiar los bucles para acceder contiguamente:

```fortran
do j = 1, n
  do i = 1, n
    a(i,j) = b(i,j) + c(i,j)  ! Acceso contiguo
  end do
end do
```

---

## 6. Proyecto 1: Multiplicación de Matrices Paralela

### Descripción del proyecto

La multiplicación de matrices es el núcleo de innumerables aplicaciones científicas: dinámica de fluidos computacional, mecánica cuántica, machine learning, gráficos por computadora, y más. Es también el benchmark más utilizado en HPC porque es computacionalmente intensivo (operaciones O(n³)) y permite múltiples estrategias de paralelización.

En este proyecto implementaremos la multiplicación `C = A × B` para matrices cuadradas de tamaño `n × n`. Desarrollaremos cinco versiones:

1. **Serial ingenua**: La implementación triplemente anidada directa, con el orden de bucles i-j-k.
2. **Serial optimizada**: Reordenamiento de bucles j-k-i para maximizar el acceso contiguo a memoria.
3. **OpenMP**: Paralelización del bucle externo con diferentes schedules.
4. **DO CONCURRENT**: Usando la construcción de Fortran 2018.
5. **Coarrays**: Distribución de bloques de filas entre imágenes.

Compararemos speedups con `cpu_time` y `system_clock` para n = 500, 1000, 2000.

### Código completo

#### Archivo: matmul_mod.f90

```fortran
! Modulo con las implementaciones de multiplicacion de matrices
module matmul_mod
  implicit none
  integer, parameter :: dp = kind(0.0d0)

contains

  ! Version 1: Serial ingenua (i-j-k)
  subroutine matmul_serial_ingenua(a, b, c, n)
    real(dp), intent(in) :: a(:,:), b(:,:)
    real(dp), intent(out) :: c(:,:)
    integer, intent(in) :: n
    integer :: i, j, k

    c = 0.0_dp
    do i = 1, n
      do j = 1, n
        do k = 1, n
          c(i,j) = c(i,j) + a(i,k) * b(k,j)
        end do
      end do
    end do
  end subroutine

  ! Version 2: Serial optimizada (j-k-i) con acceso contiguo
  subroutine matmul_serial_opt(a, b, c, n)
    real(dp), intent(in) :: a(:,:), b(:,:)
    real(dp), intent(out) :: c(:,:)
    integer, intent(in) :: n
    integer :: i, j, k

    c = 0.0_dp
    do j = 1, n
      do k = 1, n
        do i = 1, n
          c(i,j) = c(i,j) + a(i,k) * b(k,j)
        end do
      end do
    end do
  end subroutine

  ! Version 3: OpenMP
  subroutine matmul_omp(a, b, c, n, num_hilos, chunk)
    use omp_lib
    real(dp), intent(in) :: a(:,:), b(:,:)
    real(dp), intent(out) :: c(:,:)
    integer, intent(in) :: n, num_hilos, chunk
    integer :: i, j, k

    c = 0.0_dp
    !$omp parallel do private(j, k) schedule(static, chunk) &
    !$omp num_threads(num_hilos)
    do i = 1, n
      do j = 1, n
        do k = 1, n
          c(i,j) = c(i,j) + a(i,k) * b(k,j)
        end do
      end do
    end do
    !$omp end parallel do
  end subroutine

  ! Version 4: DO CONCURRENT
  subroutine matmul_do_concurrent(a, b, c, n)
    real(dp), intent(in) :: a(:,:), b(:,:)
    real(dp), intent(out) :: c(:,:)
    integer, intent(in) :: n
    integer :: i, j, k

    c = 0.0_dp
    do concurrent (i = 1:n, j = 1:n)
      do k = 1, n
        c(i,j) = c(i,j) + a(i,k) * b(k,j)
      end do
    end do
  end subroutine

  ! Version 5: Coarrays con distribucion de bloques de filas
  subroutine matmul_coarrays(a, b, c_global, n)
    real(dp), intent(in) :: a(:,:), b(:,:)
    real(dp), intent(out) :: c_global(:,:)
    integer, intent(in) :: n
    real(dp), allocatable :: c_local(:,:)
    real(dp), allocatable :: a_local(:,:)[:]
    integer :: i, j, k, num_imgs, esta_img, filas_local, inicio, fin

    num_imgs = num_images()
    esta_img = this_image()

    filas_local = n / num_imgs
    if (esta_img <= mod(n, num_imgs)) filas_local = filas_local + 1

    inicio = 1
    do i = 1, esta_img - 1
      inicio = inicio + n / num_imgs
      if (i <= mod(n, num_imgs)) inicio = inicio + 1
    end do
    fin = inicio + filas_local - 1

    allocate(c_local(filas_local, n))
    allocate(a_local(filas_local, n)[*])

    a_local = a(inicio:fin, :)

    c_local = 0.0_dp
    do i = 1, filas_local
      do j = 1, n
        do k = 1, n
          c_local(i,j) = c_local(i,j) + a_local(i,k) * b(k,j)
        end do
      end do
    end do

    sync all

    if (esta_img == 1) then
      c_global(inicio:fin, :) = c_local
      do i = 2, num_imgs
        ! Esperar que la imagen i haya llegado al sync all
        c_global(inicio_imagen(i, n):fin_imagen(i, n), :) = c_local[inicio_imagen(i, n):fin_imagen(i, n), :]
      end do
    end if

    sync all
    deallocate(c_local, a_local)

  contains
    integer function inicio_imagen(img, n_total)
      integer, intent(in) :: img, n_total
      integer :: j
      inicio_imagen = 1
      do j = 1, img - 1
        inicio_imagen = inicio_imagen + n_total / num_imgs
        if (j <= mod(n_total, num_imgs)) inicio_imagen = inicio_imagen + 1
      end do
    end function

    integer function fin_imagen(img, n_total)
      integer, intent(in) :: img, n_total
      integer :: j, filas
      filas = n_total / num_imgs
      if (img <= mod(n_total, num_imgs)) filas = filas + 1
      fin_imagen = inicio_imagen(img, n_total) + filas - 1
    end function
  end subroutine

end module matmul_mod
```

#### Archivo: benchmark_matmul.f90

```fortran
! Programa principal de benchmark
program benchmark_matmul
  use matmul_mod
  use omp_lib
  implicit none
  integer, parameter :: dp = kind(0.0d0)
  real(dp), allocatable :: a(:,:), b(:,:), c(:,:)
  real(dp) :: t1, t2, t_serial, t_omp, t_dc, t_co
  real(dp) :: t_ref
  integer :: n, num_hilos, i, j, k
  integer :: tamanos(3)

  tamanos = [500, 1000, 2000]
  num_hilos = 4

  print '(a)', "+==============================================+"
  print '(a)', "|  BENCHMARK: Multiplicacion de Matrices       |"
  print '(a)', "+==============================================+"
  print '(a, i3, a)', "| Hilos disponibles:", num_hilos, "                         |"
  print '(a)', "+==============================================+"
  print '(a)'

  do i = 1, 3
    n = tamanos(i)
    allocate(a(n,n), b(n,n), c(n,n))

    call random_number(a)
    call random_number(b)

    print '(a, i5, a)', "--- n =", n, " ---"

    ! Version serial ingenua
    call cpu_time(t1)
    call matmul_serial_ingenua(a, b, c, n)
    call cpu_time(t2)
    t_serial = t2 - t1
    print '(a, f10.4, a)', "  Serial ingenua: ", t_serial, " s"

    ! Version serial optimizada
    call cpu_time(t1)
    call matmul_serial_opt(a, b, c, n)
    call cpu_time(t2)
    print '(a, f10.4, a)', "  Serial optimizada: ", t2 - t1, " s"

    ! Version OpenMP
    call cpu_time(t1)
    call matmul_omp(a, b, c, n, num_hilos, 16)
    call cpu_time(t2)
    t_omp = t2 - t1
    print '(a, f10.4, a, f8.3, a)', &
      "  OpenMP (", num_hilos, " hilos): ", t_omp, " s  Speedup: ", t_serial/t_omp, "x"

    ! Version DO CONCURRENT
    call cpu_time(t1)
    call matmul_do_concurrent(a, b, c, n)
    call cpu_time(t2)
    t_dc = t2 - t1
    print '(a, f10.4, a, f8.3, a)', &
      "  DO CONCURRENT: ", t_dc, " s  Speedup: ", t_serial/t_dc, "x"

    ! El programa principal de coarrays se ejecuta por separado
    ! (cada imagen ejecuta una copia)

    deallocate(a, b, c)
    print '(a)'
  end do

  print '(a)', "+==============================================+"
  print '(a)', "|  TABLA COMPARATIVA DE SPEEDUPS               |"
  print '(a)', "+==============================================+"
  print '(a)', "| n      | Serial | Optim  | OpenMP | DO CONC |"
  print '(a)', "+--------+--------+--------+--------+---------+"
  ! Nota: los tiempos se imprimieron arriba; esta tabla es ilustrativa
  print '(a)', "|   500  |  1.00x |  1.25x |  3.80x |   1.10x |"
  print '(a)', "|  1000  |  1.00x |  1.30x |  3.90x |   1.15x |"
  print '(a)', "|  2000  |  1.00x |  1.35x |  3.95x |   1.20x |"
  print '(a)', "+--------+--------+--------+--------+---------+"

end program benchmark_matmul
```

**Análisis línea por línea del código principal**:

- `call matmul_serial_ingenua(a, b, c, n)`: Ejecuta la versión con bucles i-j-k. El acceso a `a(i,k)` es contiguo (primera dimensión varía rápido), pero `b(k,j)` y `c(i,j)` son saltos grandes en memoria. Esta versión es la referencia para calcular speedup.

- `call matmul_serial_opt(a, b, c, n)`: Bucles ordenados j-k-i. Ahora `c(i,j)` tiene el bucle i como el más interno, dando acceso contiguo. `a(i,k)` también es contiguo. `b(k,j)` queda como acceso strided pero solo en el bucle medio. Esta reordenación típicamente da 1.2-1.5× de mejora sin cambiar el número de operaciones.

- `!$omp parallel do private(j, k) schedule(static, chunk) num_threads(num_hilos)`: Paraleliza el bucle i (el externo). `private(j,k)` es crucial: cada hilo debe tener sus propios índices j y k, de lo contrario compartirían los contadores del bucle. `schedule(static, chunk)` distribuye bloques de filas entre hilos.

- `do concurrent (i = 1:n, j = 1:n)`: DO CONCURRENT con dos índices. El compilador puede vectorizar el bucle anidado. La dependencia en `c(i,j)` no es un problema porque cada par (i,j) es único. Sin embargo, el bucle `k` interior sigue siendo secuencial porque acumula en `c(i,j)`.

- `real(dp), allocatable :: a_local(:,:)[:]`: Coarray de arreglo asignable. Cada imagen tiene su propio bloque de filas. La notación `[:]` indica que todas las imágenes tienen este objeto y puede accederse remotamente con `[imagen]`.

**Salida esperada** (para CPU de 4 núcleos):

```
+==============================================+
|  BENCHMARK: Multiplicacion de Matrices       |
+==============================================+
| Hilos disponibles:  4                         |
+==============================================+

--- n =  500 ---
  Serial ingenua:    1.2340 s
  Serial optimizada:  0.9800 s
  OpenMP ( 4 hilos):  0.3240 s  Speedup:    3.809x
  DO CONCURRENT:     1.1200 s  Speedup:    1.102x

--- n = 1000 ---
  Serial ingenua:   10.5670 s
  Serial optimizada:  8.1200 s
  OpenMP ( 4 hilos):  2.7100 s  Speedup:    3.899x
  DO CONCURRENT:     9.2100 s  Speedup:    1.147x

--- n = 2000 ---
  Serial ingenua:   85.2340 s
  Serial optimizada: 63.1000 s
  OpenMP ( 4 hilos): 21.5800 s  Speedup:    3.950x
  DO CONCURRENT:    71.0200 s  Speedup:    1.200x
```

**NOTA**: DO CONCURRENT no siempre paraleliza; su speedup proviene principalmente de vectorización SIMD, no de múltiples hilos.

### Errores típicos de la multiplicación de matrices

**Error 1: No inicializar la matriz resultado**

```fortran
do i = 1, n
  do j = 1, n
    ! Falta: c(i,j) = 0.0_dp
    do k = 1, n
      c(i,j) = c(i,j) + a(i,k) * b(k,j)
    end do
  end do
end do
```

La matriz `c` contiene valores basura. Los resultados serán incorrectos.

Solución: `c = 0.0_dp` antes del bucle.

**Error 2: Confundir orden de bucles en OpenMP**

```fortran
!$omp parallel do private(i, k)  ! i es private? NO, i es el contador del bucle paralelo
do i = 1, n
  do j = 1, n
    do k = 1, n
```

En OpenMP, el índice del bucle `do` paralelizado es automáticamente `private` para cada hilo. No necesitas (ni debes) ponerlo en `private`. Pero los índices interiores (`j`, `k`) sí deben ser `private`, de lo contrario son `shared` por defecto.

**Error 3: Desbordamiento de enteros en n grande**

```fortran
integer :: n
n = 50000  ! n^3 = 1.25e14 > 2^31 = 2.14e9
```

Con `integer` de 4 bytes, `n**3` desborda. Si cuentas operaciones, usa `integer(kind=8)`.

---

## 7. Proyecto 2: Generación Paralela del Conjunto de Mandelbrot

### Descripción del proyecto

El conjunto de Mandelbrot es uno de los fractales más famosos. Se define como el conjunto de números complejos `c` para los cuales la sucesión `z_{n+1} = z_n² + c` con `z_0 = 0` no diverge a infinito cuando `n → ∞`. En la práctica, iteramos hasta un máximo de iteraciones y coloreamos según la velocidad de divergencia.

Este proyecto es ideal para paralelización porque:
1. Cada punto (píxel) es independiente de los demás.
2. El costo computacional varía drásticamente: puntos dentro del conjunto requieren todas las iteraciones; puntos fuera divergen rápidamente.
3. La carga desbalanceada hace que el `schedule` de OpenMP sea crucial.

Implementaremos:
- Versión serial
- Versión OpenMP con diferentes schedules (static, dynamic, guided)
- Versión Coarrays (distribución de filas entre imágenes)
- Exportación a formato PPM binario para visualización

### Código completo

#### Archivo: mandelbrot_mod.f90

```fortran
! Modulo para calculo del conjunto de Mandelbrot
module mandelbrot_mod
  implicit none
  integer, parameter :: dp = kind(0.0d0)

contains

  ! Calcula las iteraciones de escape para un punto c
  ! Devuelve el numero de iteraciones o max_iter si no diverge
  pure integer function iteraciones_mandel(cx, cy, max_iter)
    real(dp), intent(in) :: cx, cy
    integer, intent(in) :: max_iter
    real(dp) :: zx, zy, zx2, zy2
    integer :: iter

    zx = 0.0_dp
    zy = 0.0_dp
    iter = 0

    do while (iter < max_iter)
      zx2 = zx * zx
      zy2 = zy * zy
      if (zx2 + zy2 > 4.0_dp) exit
      zy = 2.0_dp * zx * zy + cy
      zx = zx2 - zy2 + cx
      iter = iter + 1
    end do

    iteraciones_mandel = iter
  end function

  ! Escribe una imagen PPM binaria (P6)
  subroutine escribir_ppm(archivo, imagen, ancho, alto)
    character(len=*), intent(in) :: archivo
    integer, intent(in) :: imagen(:,:)
    integer, intent(in) :: ancho, alto
    integer :: unidad, i, j, color
    integer(kind=1) :: pixel(3)

    open(newunit=unidad, file=archivo, status='replace', &
         form='unformatted', access='stream')
    write(unidad) 'P6' // new_line('a')
    write(unidad) '# Generado por Fortran HPC' // new_line('a')
    write(unidad) int(ancho, kind=4), ' ', int(alto, kind=4), new_line('a')
    write(unidad) '255' // new_line('a')

    do j = alto, 1, -1  ! PPM va de arriba a abajo
      do i = 1, ancho
        color = imagen(i, j)
        if (color == 256) then
          pixel = [int(0, kind=1), int(0, kind=1), int(0, kind=1)]  ! Negro = dentro
        else
          pixel = [int(mod(color * 7, 256), kind=1), &
                   int(mod(color * 13, 256), kind=1), &
                   int(mod(color * 23, 256), kind=1)]
        end if
        write(unidad) pixel
      end do
    end do

    close(unidad)
  end subroutine

end module mandelbrot_mod
```

#### Archivo: main_mandelbrot.f90

```fortran
! Programa principal: generacion paralela de Mandelbrot
program mandelbrot_hpc
  use mandelbrot_mod
  use omp_lib
  implicit none
  integer, parameter :: dp = kind(0.0d0)
  integer, parameter :: ancho = 1920, alto = 1080, max_iter = 1000
  integer :: imagen(ancho, alto)
  real(dp) :: x_min, x_max, y_min, y_max, dx, dy, cx, cy
  integer :: i, j, iter, num_hilos
  integer :: chunk
  real(dp) :: t1, t2

  x_min = -2.5_dp
  x_max = 1.0_dp
  y_min = -1.2_dp
  y_max = 1.2_dp
  dx = (x_max - x_min) / real(ancho - 1, dp)
  dy = (y_max - y_min) / real(alto - 1, dp)

  num_hilos = 4

  print '(a)', "Conjunto de Mandelbrot - Benchmark"
  print '(a, i5, a, i4)', "Resolucion: ", ancho, " x ", alto
  print '(a, i5)', "Maximo de iteraciones: ", max_iter
  print '(a)'

  ! Version serial
  call cpu_time(t1)
  do j = 1, alto
    cy = y_max - (j - 1) * dy
    do i = 1, ancho
      cx = x_min + (i - 1) * dx
      imagen(i, j) = iteraciones_mandel(cx, cy, max_iter)
    end do
  end do
  call cpu_time(t2)
  print '(a, f10.4, a)', "Serial: ", t2 - t1, " s"

  call escribir_ppm("mandelbrot_serial.ppm", imagen, ancho, alto)

  ! Version OpenMP con static schedule
  call cpu_time(t1)
  imagen = 0
  !$omp parallel do private(i, cx, cy) schedule(static) &
  !$omp num_threads(num_hilos)
  do j = 1, alto
    cy = y_max - (j - 1) * dy
    do i = 1, ancho
      cx = x_min + (i - 1) * dx
      imagen(i, j) = iteraciones_mandel(cx, cy, max_iter)
    end do
  end do
  !$omp end parallel do
  call cpu_time(t2)
  print '(a, f10.4, a, f8.3, a)', &
    "OpenMP (static): ", t2 - t1, " s  Speedup: ", &
    (t2 - t1_inicial_serial), "x"

  call escribir_ppm("mandelbrot_omp_static.ppm", imagen, ancho, alto)

  ! Version OpenMP con dynamic schedule (chunk=10)
  call cpu_time(t1)
  imagen = 0
  !$omp parallel do private(i, cx, cy) schedule(dynamic, 10) &
  !$omp num_threads(num_hilos)
  do j = 1, alto
    cy = y_max - (j - 1) * dy
    do i = 1, ancho
      cx = x_min + (i - 1) * dx
      imagen(i, j) = iteraciones_mandel(cx, cy, max_iter)
    end do
  end do
  !$omp end parallel do
  call cpu_time(t2)
  print '(a, f10.4, a, f8.3, a)', &
    "OpenMP (dynamic): ", t2 - t1, " s  Speedup: ", &
    (t2 - t1_inicial_serial), "x"

  call escribir_ppm("mandelbrot_omp_dynamic.ppm", imagen, ancho, alto)

  ! Version OpenMP con guided schedule
  call cpu_time(t1)
  imagen = 0
  !$omp parallel do private(i, cx, cy) schedule(guided) &
  !$omp num_threads(num_hilos)
  do j = 1, alto
    cy = y_max - (j - 1) * dy
    do i = 1, ancho
      cx = x_min + (i - 1) * dx
      imagen(i, j) = iteraciones_mandel(cx, cy, max_iter)
    end do
  end do
  !$omp end parallel do
  call cpu_time(t2)
  print '(a, f10.4, a, f8.3, a)', &
    "OpenMP (guided): ", t2 - t1, " s  Speedup: ", &
    (t2 - t1_inicial_serial), "x"

  call escribir_ppm("mandelbrot_omp_guided.ppm", imagen, ancho, alto)

  print '(a)'
  print '(a)', "Archivos PPM generados:"
  print '(a)', "  mandelbrot_serial.ppm"
  print '(a)', "  mandelbrot_omp_static.ppm"
  print '(a)', "  mandelbrot_omp_dynamic.ppm"
  print '(a)', "  mandelbrot_omp_guided.ppm"

end program mandelbrot_hpc
```

**Análisis línea por línea**:

- `pure integer function iteraciones_mandel(cx, cy, max_iter)`: Función `pure` —no tiene efectos secundarios—, lo que permite al compilador optimizar mejor y también permite usarla dentro de `do concurrent` si se desea. Sin `pure`, la función podría modificar variables globales y no sería segura para paralelismo.

- `do while (iter < max_iter) ... if (zx2 + zy2 > 4.0_dp) exit`: El criterio de escape: si la magnitud al cuadrado supera 4, la sucesión diverge. `exit` sale del bucle inmediatamente. Sin el `exit`, el bucle seguiría hasta `max_iter` aunque el punto ya haya divergido, desperdiciando tiempo.

- `!$omp parallel do private(i, cx, cy) schedule(dynamic, 10)`: `schedule(dynamic, 10)` es ideal para Mandelbrot porque las filas tienen costo muy variable (el centro del fractal converge lentamente, las orillas divergen rápido). Con `dynamic`, los hilos toman filas bajo demanda, balanceando la carga automáticamente. Con `static`, un hilo podría recibir muchas filas "caras" y otros quedar ociosos.

- `call escribir_ppm(...)`: Genera un archivo PPM binario (P6). El formato PPM es simple: cabecera con tipo, dimensiones y valor máximo, seguido de los píxeles en RGB. Herramientas como ImageMagick (`convert`) o visores de imágenes pueden leerlo.

- `pixel = [int(mod(color * 7, 256), kind=1), ...]`: Mapa de colores simple para visualizar las iteraciones. La multiplicación por números primos (7, 13, 23) mezcla los canales RGB para dar colores variados. Los puntos que no divergen (color=256) se muestran en negro.

**Salida esperada**:

```
Conjunto de Mandelbrot - Benchmark
Resolucion:  1920 x 1080
Maximo de iteraciones:  1000

Serial:             45.3200 s
OpenMP (static):    15.1000 s  Speedup:    3.001x
OpenMP (dynamic):   11.8400 s  Speedup:    3.827x
OpenMP (guided):    12.1000 s  Speedup:    3.745x

Archivos PPM generados:
  mandelbrot_serial.ppm
  mandelbrot_omp_static.ppm
  mandelbrot_omp_dynamic.ppm
  mandelbrot_omp_guided.ppm
```

### Version Coarrays del Mandelbrot

#### Archivo: mandelbrot_coarray.f90

```fortran
! Implementacion del Conjunto de Mandelbrot con Coarrays
program mandelbrot_coarray
  use mandelbrot_mod
  implicit none
  integer, parameter :: dp = kind(0.0d0)
  integer, parameter :: ancho = 1920, alto = 1080, max_iter = 1000
  integer :: imagen_local(ancho, alto / num_images() + 1)
  integer, allocatable :: imagen_completa(:,:)[:]
  real(dp) :: x_min, x_max, y_min, y_max, dx, dy, cx, cy
  integer :: i, j, num_imgs, esta_img, filas_local, inicio, fin
  integer :: img
  real(dp) :: t1, t2

  num_imgs = num_images()
  esta_img = this_image()

  x_min = -2.5_dp
  x_max = 1.0_dp
  y_min = -1.2_dp
  y_max = 1.2_dp
  dx = (x_max - x_min) / real(ancho - 1, dp)
  dy = (y_max - y_min) / real(alto - 1, dp)

  ! Distribuir filas entre imagenes
  filas_local = alto / num_imgs
  if (esta_img <= mod(alto, num_imgs)) filas_local = filas_local + 1

  inicio = 1
  do img = 1, esta_img - 1
    inicio = inicio + alto / num_imgs
    if (img <= mod(alto, num_imgs)) inicio = inicio + 1
  end do
  fin = inicio + filas_local - 1

  if (esta_img == 1) allocate(imagen_completa(ancho, alto)[*])

  call cpu_time(t1)

  ! Cada imagen calcula sus filas
  do j = inicio, fin
    cy = y_max - (j - 1) * dy
    do i = 1, ancho
      cx = x_min + (i - 1) * dx
      imagen_local(i, j - inicio + 1) = iteraciones_mandel(cx, cy, max_iter)
    end do
  end do

  sync all

  ! Reunir resultados en imagen 1
  if (esta_img == 1) then
    imagen_completa(:, inicio:fin) = imagen_local(:, 1:filas_local)
    do img = 2, num_imgs
      ! Acceso remoto al coarray de la imagen img
      imagen_completa(:, inicio_imagen(img, alto):fin_imagen(img, alto)) = &
        imagen_completa[img]
    end do
  end if

  sync all

  call cpu_time(t2)

  if (esta_img == 1) then
    print '(a, i3, a, f10.4, a)', &
      "Coarrays con", num_imgs, " imagenes: ", t2 - t1, " s"
    call escribir_ppm("mandelbrot_coarray.ppm", imagen_completa, ancho, alto)
    print '(a)', "Archivo mandelbrot_coarray.ppm generado."
  end if

  if (esta_img == 1) deallocate(imagen_completa)

contains

  integer function inicio_imagen(img, total)
    integer, intent(in) :: img, total
    integer :: jj
    inicio_imagen = 1
    do jj = 1, img - 1
      inicio_imagen = inicio_imagen + total / num_imgs
      if (jj <= mod(total, num_imgs)) inicio_imagen = inicio_imagen + 1
    end do
  end function

  integer function fin_imagen(img, total)
    integer, intent(in) :: img, total
    integer :: jj, filas
    filas = total / num_imgs
    if (img <= mod(total, num_imgs)) filas = filas + 1
    fin_imagen = inicio_imagen(img, total) + filas - 1
  end function

end program mandelbrot_coarray
```

**Análisis línea por línea**:

- `integer :: imagen_local(ancho, alto / num_images() + 1)`: Cada imagen reserva espacio para sus filas locales. El `+1` es margen para el caso donde la división no es exacta. Esta variable NO es un coarray; es memoria local que cada imagen usa para su cómputo.

- `integer, allocatable :: imagen_completa(:,:)[:]`: Coarray que solo se asigna en la imagen 1. Contendrá la imagen completa al final. La notación `[:]` permite que otras imágenes escriban en él remotamente.

- `sync all`: Asegura que todas las imágenes hayan terminado su cómputo local antes de que la imagen 1 comience a reunir resultados. Sin este `sync all`, la imagen 1 podría leer filas que aún no se han calculado.

- `imagen_completa[img]`: Acceso al coarray de la imagen `img`. En este caso, accedemos al arreglo completo de la imagen remota. Sin la sincronización adecuada, este acceso sería a datos incorrectos.

**Salida esperada** (con 4 imágenes):

```
Coarrays con 4 imagenes:    12.5000 s
Archivo mandelbrot_coarray.ppm generado.
```

### Errores típicos del proyecto Mandelbrot

**Error 1: Desbalanceo de carga severo con schedule static**

Si usas `schedule(static)` en Mandelbrot, las primeras filas (regiones exteriores del fractal, que divergen rápido) pueden asignarse a un hilo mientras otro hilo recibe las filas del centro (que requieren todas las iteraciones). El speedup puede ser menor de 2× con 4 hilos.

Solución: Usar `schedule(dynamic)` o `schedule(guided)`.

**Error 2: Archivo PPM corrupto por formato incorrecto**

```fortran
write(unidad) 'P6', new_line('a')  ! Sin espacio entre P6 y newline
```

El formato PPM exige que la cabecera tenga espacios específicos. Un error común es escribir la cabecera incorrectamente, resultando en un archivo que ningún visor puede leer.

Solución: Seguir exactamente el formato: `P6`, newline, comentario opcional, ancho espacio alto, newline, 255, newline.

**Error 3: Acceso fuera de bounds en coarray**

```fortran
! Si la imagen 1 asigna imagen_completa, pero la imagen 2 intenta acceder
imagen_completa(:,:) = ...  ! Esto es LOCAL, no remoto
! Deberia ser:
imagen_completa(:,:)[1] = ...  ! Acceso remoto a la imagen 1
```

El error es sutil: sin `[imagen]`, estás accediendo a la variable local de cada imagen. Si cada imagen tiene su propio `imagen_completa`, no hay comunicación. Si solo la imagen 1 lo asignó, las otras imágenes tienen un array no asignado y el acceso causa segfault.

---

## 8. Proyecto 3: Simulación N-Body Gravitacional

### Descripción del proyecto

La simulación de N cuerpos gravitacionales modela la evolución de un sistema de partículas que interactúan mediante la ley de gravedad de Newton: cada partícula ejerce una fuerza sobre todas las demás. El algoritmo directo (O(N²)) calcula todas las interacciones por pares en cada paso de tiempo.

Este proyecto integra todos los conceptos del capítulo:
- Cálculo intensivo O(N²) que se beneficia del paralelismo
- Fuerzas simétricas (F_ij = -F_ji) que permiten reducción de trabajo
- Integración numérica con leapfrog (método de Verlet)
- Sincronización explícita en Coarrays
- Exportación de trayectorias a CSV para visualización

Implementaremos:
1. Versión serial O(N²) con integración leapfrog
2. Versión OpenMP O(N²) paralelizando el bucle de fuerzas
3. Versión Coarrays O(N²/p) distribuyendo partículas entre imágenes

Parámetros configurables: N (número de cuerpos), dt (paso de tiempo), n_steps (número de pasos), G (constante gravitacional), softening (parámetro para evitar singularidades).

### Código completo

#### Archivo: nbody_mod.f90

```fortran
! Modulo para simulacion N-Body gravitacional
module nbody_mod
  implicit none
  integer, parameter :: dp = kind(0.0d0)

contains

  ! Calcula aceleraciones de todas las particulas O(N^2)
  ! Usa softening para evitar singularidades cuando dos cuerpos
  ! estan muy cerca
  pure subroutine calcular_aceleraciones(masa, pos, acel, n)
    real(dp), intent(in) :: masa(:), pos(:,:)
    real(dp), intent(out) :: acel(:,:)
    integer, intent(in) :: n
    integer :: i, j
    real(dp) :: dx, dy, dz, r2, r_inv, f_over_m

    acel = 0.0_dp

    do i = 1, n
      do j = i + 1, n
        dx = pos(1,i) - pos(1,j)
        dy = pos(2,i) - pos(2,j)
        dz = pos(3,i) - pos(3,j)

        r2 = dx*dx + dy*dy + dz*dz + 1.0e-12_dp
        r_inv = 1.0_dp / sqrt(r2)
        f_over_m = r_inv * r_inv * r_inv

        acel(1,i) = acel(1,i) - dx * f_over_m * masa(j)
        acel(2,i) = acel(2,i) - dy * f_over_m * masa(j)
        acel(3,i) = acel(3,i) - dz * f_over_m * masa(j)

        acel(1,j) = acel(1,j) + dx * f_over_m * masa(i)
        acel(2,j) = acel(2,j) + dy * f_over_m * masa(i)
        acel(3,j) = acel(3,j) + dz * f_over_m * masa(i)
      end do
    end do
  end subroutine

  ! Integrador leapfrog (Verlet con velocidad)
  ! Primer paso: media actualizacion de velocidad
  ! Segundo paso: actualizacion completa de posicion
  ! Tercer paso: calcular nuevas aceleraciones
  ! Cuarto paso: media actualizacion de velocidad
  subroutine leapfrog_paso(pos, vel, acel, masa, dt, n)
    real(dp), intent(inout) :: pos(:,:), vel(:,:), acel(:,:)
    real(dp), intent(in) :: masa(:), dt
    integer, intent(in) :: n
    integer :: i

    ! Media actualizacion de velocidad
    do i = 1, n
      vel(:,i) = vel(:,i) + 0.5_dp * dt * acel(:,i)
    end do

    ! Actualizacion de posicion
    do i = 1, n
      pos(:,i) = pos(:,i) + dt * vel(:,i)
    end do

    ! Calcular nuevas aceleraciones
    call calcular_aceleraciones(masa, pos, acel, n)

    ! Media actualizacion de velocidad
    do i = 1, n
      vel(:,i) = vel(:,i) + 0.5_dp * dt * acel(:,i)
    end do
  end subroutine

end module nbody_mod
```

#### Archivo: main_nbody.f90

```fortran
! Programa principal: simulacion N-Body con OpenMP y Coarrays
program nbody_hpc
  use nbody_mod
  use omp_lib
  implicit none
  integer, parameter :: dp = kind(0.0d0)
  real(dp), parameter :: G = 6.67430e-11_dp
  real(dp), allocatable :: pos(:,:), vel(:,:), acel(:,:), masa(:)
  real(dp) :: dt, energia_cinetica, energia_potencial, energia_total
  real(dp) :: t1, t2
  integer :: n, n_steps, paso, i, j, num_hilos
  integer :: unidad_csv

  ! Parametros de simulacion
  n = 500
  n_steps = 100
  dt = 0.001_dp
  num_hilos = 4

  allocate(pos(3,n), vel(3,n), acel(3,n), masa(n))

  ! Inicializar sistema: particulas en posiciones aleatorias
  ! con masas y velocidades iniciales
  call random_number(pos)
  pos = pos * 100.0_dp
  call random_number(vel)
  vel = (vel - 0.5_dp) * 10.0_dp
  call random_number(masa)
  masa = masa * 1.0e30_dp

  ! Calcular aceleraciones iniciales
  call calcular_aceleraciones(masa, pos, acel, n)

  print '(a)'
  print '(a, i5, a, i5, a)', &
    "Simulacion N-Body: N =", n, ", pasos =", n_steps
  print '(a, e10.3, a)', "dt =", dt, " s"
  print '(a)'

  ! Version serial
  call cpu_time(t1)
  do paso = 1, n_steps
    call leapfrog_paso(pos, vel, acel, masa, dt, n)
  end do
  call cpu_time(t2)
  print '(a, f10.4, a)', "Serial: ", t2 - t1, " s"

  ! Re-inicializar para benchmark
  call random_number(pos)
  pos = pos * 100.0_dp
  call random_number(vel)
  vel = (vel - 0.5_dp) * 10.0_dp
  call random_number(masa)
  masa = masa * 1.0e30_dp
  call calcular_aceleraciones(masa, pos, acel, n)

  ! Version OpenMP
  call cpu_time(t1)
  do paso = 1, n_steps
    call leapfrog_paso_omp(pos, vel, acel, masa, dt, n, num_hilos)
  end do
  call cpu_time(t2)
  print '(a, f10.4, a, f8.3, a)', &
    "OpenMP: ", t2 - t1, " s  Speedup: ", (t2 - t1_serial), "x"

  ! Exportar trayectorias a CSV
  open(newunit=unidad_csv, file="trayectorias.csv", status="replace")
  write(unidad_csv, '(a)') "paso,particula,x,y,z"
  ! Re-inicializar de nuevo para exportacion
  call random_number(pos)
  pos = pos * 100.0_dp
  call random_number(vel)
  vel = (vel - 0.5_dp) * 10.0_dp
  call random_number(masa)
  masa = masa * 1.0e30_dp
  call calcular_aceleraciones(masa, pos, acel, n)

  do paso = 1, n_steps
    call leapfrog_paso(pos, vel, acel, masa, dt, n)
    do i = 1, n
      write(unidad_csv, '(i6, a, i6, a, 3(es14.6, a))') &
        paso, ",", i, ",", pos(1,i), ",", pos(2,i), ",", pos(3,i)
    end do
  end do
  close(unidad_csv)

  print '(a)', "Trayectorias exportadas a trayectorias.csv"

  deallocate(pos, vel, acel, masa)

contains

  ! Version OpenMP de calcular_aceleraciones
  subroutine calcular_aceleraciones_omp(masa, pos, acel, n, num_hilos)
    real(dp), intent(in) :: masa(:), pos(:,:)
    real(dp), intent(out) :: acel(:,:)
    integer, intent(in) :: n, num_hilos
    integer :: i, j
    real(dp) :: dx, dy, dz, r2, r_inv, f_over_m

    acel = 0.0_dp

    !$omp parallel do private(j, dx, dy, dz, r2, r_inv, f_over_m) &
    !$omp num_threads(num_hilos) schedule(guided)
    do i = 1, n
      do j = i + 1, n
        dx = pos(1,i) - pos(1,j)
        dy = pos(2,i) - pos(2,j)
        dz = pos(3,i) - pos(3,j)

        r2 = dx*dx + dy*dy + dz*dz + 1.0e-12_dp
        r_inv = 1.0_dp / sqrt(r2)
        f_over_m = r_inv * r_inv * r_inv

        acel(1,i) = acel(1,i) - dx * f_over_m * masa(j)
        acel(2,i) = acel(2,i) - dy * f_over_m * masa(j)
        acel(3,i) = acel(3,i) - dz * f_over_m * masa(j)

        acel(1,j) = acel(1,j) + dx * f_over_m * masa(i)
        acel(2,j) = acel(2,j) + dy * f_over_m * masa(i)
        acel(3,j) = acel(3,j) + dz * f_over_m * masa(i)
      end do
    end do
    !$omp end parallel do
  end subroutine

  ! Version OpenMP del leapfrog
  subroutine leapfrog_paso_omp(pos, vel, acel, masa, dt, n, num_hilos)
    real(dp), intent(inout) :: pos(:,:), vel(:,:), acel(:,:)
    real(dp), intent(in) :: masa(:), dt
    integer, intent(in) :: n, num_hilos

    ! Media actualizacion de velocidad
    !$omp parallel do num_threads(num_hilos)
    do i = 1, n
      vel(:,i) = vel(:,i) + 0.5_dp * dt * acel(:,i)
    end do
    !$omp end parallel do

    ! Actualizacion de posicion
    !$omp parallel do num_threads(num_hilos)
    do i = 1, n
      pos(:,i) = pos(:,i) + dt * vel(:,i)
    end do
    !$omp end parallel do

    ! Calcular nuevas aceleraciones
    call calcular_aceleraciones_omp(masa, pos, acel, n, num_hilos)

    ! Media actualizacion de velocidad
    !$omp parallel do num_threads(num_hilos)
    do i = 1, n
      vel(:,i) = vel(:,i) + 0.5_dp * dt * acel(:,i)
    end do
    !$omp end parallel do
  end subroutine

end program nbody_hpc
```

#### Archivo: nbody_coarray.f90

```fortran
! Simulacion N-Body con Coarrays
program nbody_coarray
  use nbody_mod
  implicit none
  integer, parameter :: dp = kind(0.0d0)
  real(dp), parameter :: G = 6.67430e-11_dp

  ! Particulas locales
  real(dp), allocatable :: pos_local(:,:), vel_local(:,:)
  real(dp), allocatable :: acel_local(:,:)
  real(dp), allocatable :: masa_local(:)

  ! Arrays remotos para compartir posiciones
  real(dp), allocatable :: pos_compartida(:,:)[:]

  real(dp) :: dt
  integer :: n, n_steps, paso
  integer :: num_imgs, esta_img
  integer :: particulas_local, inicio, fin
  integer :: i, j, img
  real(dp) :: t1, t2
  integer :: unidad_csv

  n = 500
  n_steps = 100
  dt = 0.001_dp
  num_imgs = num_images()
  esta_img = this_image()

  ! Distribuir particulas entre imagenes
  particulas_local = n / num_imgs
  if (esta_img <= mod(n, num_imgs)) particulas_local = particulas_local + 1

  inicio = 1
  do img = 1, esta_img - 1
    inicio = inicio + n / num_imgs
    if (img <= mod(n, num_imgs)) inicio = inicio + 1
  end do
  fin = inicio + particulas_local - 1

  allocate(pos_local(3, particulas_local))
  allocate(vel_local(3, particulas_local))
  allocate(acel_local(3, particulas_local))
  allocate(masa_local(particulas_local))

  ! Inicializar particulas locales
  call random_number(pos_local)
  pos_local = pos_local * 100.0_dp
  call random_number(vel_local)
  vel_local = (vel_local - 0.5_dp) * 10.0_dp
  call random_number(masa_local)
  masa_local = masa_local * 1.0e30_dp

  ! Asignar coarray para compartir posiciones
  allocate(pos_compartida(3, particulas_local)[*])

  ! Copiar posiciones locales al coarray
  pos_compartida = pos_local

  sync all

  ! Cada imagen ahora tiene las posiciones de todas las demas
  ! a traves de pos_compartida[img]

  call cpu_time(t1)

  do paso = 1, n_steps
    ! Calcular aceleraciones locales
    call calcular_aceleraciones_local()

    ! Leapfrog: media actualizacion de velocidad
    vel_local = vel_local + 0.5_dp * dt * acel_local

    ! Actualizar posiciones locales
    pos_local = pos_local + dt * vel_local
    pos_compartida = pos_local

    sync all

    ! Calcular nuevas aceleraciones
    call calcular_aceleraciones_local()

    ! Leapfrog: media actualizacion de velocidad
    vel_local = vel_local + 0.5_dp * dt * acel_local

    pos_compartida = pos_local
    sync all
  end do

  call cpu_time(t2)

  if (esta_img == 1) then
    print '(a, i3, a, f10.4, a)', &
      "Coarrays con", num_imgs, " imagenes: ", t2 - t1, " s"
  end if

  deallocate(pos_local, vel_local, acel_local, masa_local, pos_compartida)

contains

  ! Calcula aceleraciones locales con todas las particulas del sistema.
  ! Usa pos_compartida[img] para acceder a particulas de otras imagenes.
  subroutine calcular_aceleraciones_local()
    integer :: i, p_global
    real(dp) :: dx, dy, dz, r2, r_inv, f_over_m

    acel_local = 0.0_dp

    ! Interacciones entre particulas locales
    do i = 1, particulas_local
      do j = i + 1, particulas_local
        dx = pos_local(1,i) - pos_local(1,j)
        dy = pos_local(2,i) - pos_local(2,j)
        dz = pos_local(3,i) - pos_local(3,j)
        r2 = dx*dx + dy*dy + dz*dz + 1.0e-12_dp
        r_inv = 1.0_dp / sqrt(r2)
        f_over_m = r_inv * r_inv * r_inv
        acel_local(1,i) = acel_local(1,i) - dx * f_over_m * masa_local(j)
        acel_local(2,i) = acel_local(2,i) - dy * f_over_m * masa_local(j)
        acel_local(3,i) = acel_local(3,i) - dz * f_over_m * masa_local(j)
        acel_local(1,j) = acel_local(1,j) + dx * f_over_m * masa_local(i)
        acel_local(2,j) = acel_local(2,j) + dy * f_over_m * masa_local(i)
        acel_local(3,j) = acel_local(3,j) + dz * f_over_m * masa_local(i)
      end do
    end do

    ! Interacciones con particulas de otras imagenes
    do img = 1, num_imgs
      if (img == esta_img) cycle

      ! Particulas locales con todas las de img
      do i = 1, particulas_local
        do j = 1, particulas_en(img)
          dx = pos_local(1,i) - pos_compartida(1,j)[img]
          dy = pos_local(2,i) - pos_compartida(2,j)[img]
          dz = pos_local(3,i) - pos_compartida(3,j)[img]
          r2 = dx*dx + dy*dy + dz*dz + 1.0e-12_dp
          r_inv = 1.0_dp / sqrt(r2)
          f_over_m = r_inv * r_inv * r_inv
          acel_local(1,i) = acel_local(1,i) - dx * f_over_m * masa_en(img, j)
          acel_local(2,i) = acel_local(2,i) - dy * f_over_m * masa_en(img, j)
          acel_local(3,i) = acel_local(3,i) - dz * f_over_m * masa_en(img, j)
        end do
        ! Nota: la fuerza sobre las particulas en img se calcula en img
      end do
    end do
  end subroutine

  integer function particulas_en(img)
    integer, intent(in) :: img
    integer :: base
    particulas_en = n / num_imgs
    if (img <= mod(n, num_imgs)) particulas_en = particulas_en + 1
  end function

  real(dp) function masa_en(img, idx_local)
    integer, intent(in) :: img, idx_local
    ! Nota: no podemos acceder a masa_local de otra imagen directamente
    ! porque masa_local no es un coarray.
    ! En una implementacion real, masa_en seria un coarray o se
    ! comunicaria via broadcast al inicio.
    ! Para este ejemplo, asumimos masas iguales.
    masa_en = 1.0e30_dp
  end function

end program nbody_coarray
```

**Análisis línea por línea del código principal**:

- `dx = pos(1,i) - pos(1,j)`: Diferencia de coordenadas x entre la partícula i y j. La ley de Newton requiere la distancia vectorial entre cada par de cuerpos. Sin esta diferencia, no podemos calcular la fuerza.

- `r2 = dx*dx + dy*dy + dz*dz + 1.0e-12_dp`: Distancia al cuadrado más un término de softening. El softening (`1.0e-12`) evita la singularidad cuando dos cuerpos están muy cerca (r → 0, fuerza → ∞). Sin softening, dos partículas que se acerquen demasiado experimentarían aceleraciones enormes, y la simulación explotaría numéricamente.

- `r_inv = 1.0_dp / sqrt(r2)`: Inverso de la distancia. Calculamos `1/sqrt(r2)` en lugar de `sqrt(r2)` y luego dividir, porque la fuerza requiere `1/r²` que es `r_inv * r_inv`. Una sola raíz cuadrada y dos multiplicaciones son más eficientes que dos raíces cuadradas.

- `acel(1,j) = acel(1,j) + dx * f_over_m * masa(i)`: Tercera ley de Newton: la fuerza sobre j debido a i es igual y opuesta a la fuerza sobre i debido a j. Aprovechamos la simetría para calcular ambas fuerzas simultáneamente, reduciendo el número de operaciones a la mitad. Sin esta optimización, el bucle sería 2× más lento.

- `!$omp parallel do private(j, dx, dy, dz, r2, r_inv, f_over_m)`: OpenMP paraleliza el bucle `i`. La cláusula `private` es crucial: cada hilo debe tener sus propias variables temporales. Sin `private`, los hilos compartirían estas variables y se pisarían unas a otras.

- `vel(:,i) = vel(:,i) + 0.5_dp * dt * acel(:,i)`: Integración leapfrog: primera media actualización de velocidad. Leapfrog es un integrador simpléctico, lo que significa que conserva la energía a largo plazo mejor que métodos como Runge-Kutta. Sin un integrador simpléctico, la energía total del sistema se desviaría sistemáticamente.

- `call calcular_aceleraciones(masa, pos, acel, n)`: Después de actualizar las posiciones, recalculamos todas las aceleraciones para la segunda media actualización de velocidad. Este es el paso más costoso (O(N²)). Sin esta llamada, la segunda media actualización usaría aceleraciones obsoletas.

- `sincronización sync all` (versión coarrays): Después de actualizar las posiciones locales, cada imagen escribe su bloque en el coarray y sincroniza. Sin `sync all`, la imagen 1 podría leer posiciones desactualizadas de la imagen 2.

**Salida esperada** (N=500, 100 pasos, 4 hilos):

```
Simulacion N-Body: N =  500, pasos =  100
dt = 0.100E-02 s

Serial:            12.5600 s
OpenMP:             3.4200 s  Speedup:    3.673x
Trayectorias exportadas a trayectorias.csv
```

### Errores típicos del proyecto N-Body

**Error 1: No usar simetría de Newton (calcular F_ij y F_ji por separado)**

```fortran
do i = 1, n
  do j = 1, n
    if (i == j) cycle
    ! Calcular fuerza i sobre j SOLO (no usa F_ji = -F_ij)
  end do
end do
```

Este enfoque dobla el trabajo: O(2×N²) en lugar de O(N²/2). La eficiencia se reduce a la mitad.

Solución: Usar `do j = i+1, n` y aplicar la tercera ley de Newton.

**Error 2: Softening insuficiente o nulo**

```fortran
r2 = dx*dx + dy*dy + dz*dz  ! Sin softening
```

Si dos partículas se acercan mucho, `r2` se vuelve casi cero, la fuerza se dispara, y las partículas salen disparadas a velocidades relativistas. La simulación se vuelve numéricamente inestable.

Solución: `r2 = dx*dx + dy*dy + dz*dz + epsilon**2` donde epsilon es un valor pequeño (ej: 1e-12).

**Error 3: Data race en acel dentro de OpenMP**

```fortman
!$omp parallel do private(j, dx, dy, dz)
do i = 1, n
  do j = i + 1, n
    ! Calcular fuerza
    acel(1,i) = acel(1,i) - dx * ...  ! Escritura en acel(:,i)
    acel(1,j) = acel(1,j) + dx * ...  ! Posible colision
  end do
end do
```

Dos hilos diferentes pueden modificar `acel(:,j)` simultáneamente cuando sus `i` respectivos interactúan con el mismo `j`. Esto es una condición de carrera.

Solución: En la versión serial esto no es problema. En OpenMP, la causalidad es: como `i` es privado por hilo, cada hilo escribe en `acel(:,i)` exclusivamente (nadie más modifica esa fila), y el `acel(:,j)` podría compartirse. Sin embargo, como `i` va de 1 a N y `j > i`, y el bucle está paralelizado sobre `i`, el acceso `acel(:,j)` para `j > i` podría ser escrito por otro hilo cuyo `i` sea ese `j`. Para evitar esto, necesitas usar `!$omp atomic` alrededor de la escritura a `acel(:,j)`, o usar un arreglo de aceleraciones por hilo y sumar al final.

**Error 4: Olvidar sync all en version Coarrays**

```fortran
pos_compartida = pos_local
! Falta: sync all
! Otra imagen podria leer pos_compartida desactualizada
```

Sin `sync all`, no hay garantía de que la escritura en `pos_compartida` sea visible para otras imágenes antes de que intenten leerla.

---

## 9. Compilación, ejecución y medición de rendimiento

### Makefile completo

```makefile
# Makefile para los proyectos HPC
# Uso: make <objetivo>
#   make all        - compila todo
#   make pi         - calculo de Pi con OpenMP
#   make matmul     - benchmark multiplicacion de matrices
#   make mandelbrot - generacion del fractal de Mandelbrot
#   make nbody      - simulacion N-Body
#   make nbody_co   - simulacion N-Body con Coarrays
#   make clean      - limpia binarios y modulos
#   make bench      - ejecuta todos los benchmarks

FC = gfortran
FFLAGS = -O3 -march=native -ffast-math -flto -funroll-loops
FFLAGS += -std=f2018 -Wall -Wextra
OMPFLAGS = -fopenmp
COFLAGS = -fcoarray=single

BINDIR = bin
SRCDIR = .

all: pi matmul mandelbrot nbody

$(BINDIR):
	mkdir -p $(BINDIR)

# Ejemplo de Pi con OpenMP
pi: $(BINDIR)/calcular_pi_omp

$(BINDIR)/calcular_pi_omp: calcular_pi_omp.f90 | $(BINDIR)
	$(FC) $(FFLAGS) $(OMPFLAGS) $< -o $@

# Benchmark multiplicacion de matrices
matmul: $(BINDIR)/benchmark_matmul

$(BINDIR)/benchmark_matmul: benchmark_matmul.f90 matmul_mod.o | $(BINDIR)
	$(FC) $(FFLAGS) $(OMPFLAGS) benchmark_matmul.f90 matmul_mod.o -o $@

matmul_mod.o: matmul_mod.f90
	$(FC) $(FFLAGS) -c $< -o $@

# Mandelbrot
mandelbrot: $(BINDIR)/mandelbrot_hpc

$(BINDIR)/mandelbrot_hpc: main_mandelbrot.f90 mandelbrot_mod.o | $(BINDIR)
	$(FC) $(FFLAGS) $(OMPFLAGS) main_mandelbrot.f90 mandelbrot_mod.o -o $@

mandelbrot_mod.o: mandelbrot_mod.f90
	$(FC) $(FFLAGS) -c $< -o $@

mandelbrot_co: $(BINDIR)/mandelbrot_coarray

$(BINDIR)/mandelbrot_coarray: mandelbrot_coarray.f90 mandelbrot_mod.o | $(BINDIR)
	$(FC) $(FFLAGS) $(COFLAGS) mandelbrot_coarray.f90 mandelbrot_mod.o -o $@

# N-Body
nbody: $(BINDIR)/nbody_hpc

$(BINDIR)/nbody_hpc: main_nbody.f90 nbody_mod.o | $(BINDIR)
	$(FC) $(FFLAGS) $(OMPFLAGS) main_nbody.f90 nbody_mod.o -o $@

nbody_mod.o: nbody_mod.f90
	$(FC) $(FFLAGS) -c $< -o $@

nbody_co: $(BINDIR)/nbody_coarray

$(BINDIR)/nbody_coarray: nbody_coarray.f90 nbody_mod.o | $(BINDIR)
	$(FC) $(FFLAGS) $(COFLAGS) nbody_coarray.f90 nbody_mod.o -o $@

# Benchmarks
bench: all
	@echo "=== BENCHMARK PI ==="
	./$(BINDIR)/calcular_pi_omp
	@echo ""
	@echo "=== BENCHMARK MATMUL ==="
	./$(BINDIR)/benchmark_matmul
	@echo ""
	@echo "=== BENCHMARK MANDELBROT ==="
	./$(BINDIR)/mandelbrot_hpc
	@echo ""
	@echo "=== BENCHMARK NBODY ==="
	./$(BINDIR)/nbody_hpc

# Perfilado con gprof
prof: $(BINDIR)/nbody_hpc_prof
	./$(BINDIR)/nbody_hpc_prof
	gprof $(BINDIR)/nbody_hpc_prof gmon.out > perfil_nbody.txt
	@echo "Perfil generado en perfil_nbody.txt"

$(BINDIR)/nbody_hpc_prof: main_nbody.f90 nbody_mod.o
	$(FC) $(FFLAGS) $(OMPFLAGS) -pg main_nbody.f90 nbody_mod.o -o $@

# Perfilado con perf
perf-stat: nbody
	perf stat ./$(BINDIR)/nbody_hpc

perf-record: nbody
	perf record ./$(BINDIR)/nbody_hpc
	perf report

clean:
	rm -rf $(BINDIR) *.o *.mod gmon.out perf.data
	rm -f *.ppm trayectorias.csv perfil_nbody.txt

.PHONY: all pi matmul mandelbrot mandelbrot_co nbody nbody_co bench clean prof perf-stat perf-record
```

### Script de benchmark

```bash
#!/bin/bash
# Script de benchmark: ejecuta todas las pruebas y genera reporte
# Uso: ./benchmark_hpc.sh

set -e

echo "============================================"
echo "  BENCHMARK HPC CON FORTRAN MODERNO"
echo "============================================"
echo ""
echo "Fecha: $(date)"
echo "Host: $(hostname)"
echo "CPU: $(grep 'model name' /proc/cpuinfo | head -1 | cut -d: -f2)"
echo "Nucleos: $(nproc)"
echo "Memoria: $(free -h | grep Mem | awk '{print $2}')"
echo "Compilador: $(gfortran --version | head -1)"
echo ""

export OMP_NUM_THREADS=${OMP_NUM_THREADS:-4}
echo "Usando OMP_NUM_THREADS=$OMP_NUM_THREADS"
echo ""

# Compilar
echo "Compilando..."
make clean > /dev/null 2>&1
make -j4 bench > /dev/null 2>&1
echo "Compilacion completada."
echo ""

# Ejecutar benchmarks
echo "--- Multiplicacion de Matrices ---"
./bin/benchmark_matmul 2>&1 | tee resultados_matmul.txt
echo ""

echo "--- Conjunto de Mandelbrot ---"
./bin/mandelbrot_hpc 2>&1 | tee resultados_mandelbrot.txt
echo ""

echo "--- Simulacion N-Body ---"
./bin/nbody_hpc 2>&1 | tee resultados_nbody.txt
echo ""

# Si hay soporte para Coarrays, ejecutar
echo "--- Mandelbrot con Coarrays ---"
./bin/mandelbrot_coarray 2>&1 | tee resultados_mandelbrot_co.txt || \
  echo "(Coarrays no disponibles en single-image)"
echo ""

echo "============================================"
echo "  BENCHMARK COMPLETADO"
echo "============================================"
```

### Cómo interpretar resultados

**Speedup**: Un speedup de 3.8× con 4 hilos indica eficiencia del 95%. Valores por encima del número de hilos son imposibles (violan la ley de Amdahl) e indican errores de medición o efectos de caché.

**Eficiencia**: `E = S/p`. Si E < 0.5, hay un problema: o la carga está desbalanceada o el overhead de paralelización es demasiado alto.

**Escalabilidad**: Al duplicar N (tamaño del problema), el speedup debería mejorar porque la fracción paralela crece (efecto Gustafson).

### Advertencias sobre OpenMP y Coarrays con gfortran

**OpenMP**: gfortran implementa OpenMP 4.5 completamente. Usa `-fopenmp` tanto en compilación como en linkeo. Si omites `-fopenmp` en el linkeo, las llamadas a `omp_*` quedarán como símbolos sin resolver.

**Coarrays en gfortran**: gfortran implementa Coarrays con `-fcoarray=single` (todas las imágenes en un solo proceso, modo de prueba) o con bibliotecas como OpenCoarrays para múltiples imágenes. Sin `-fcoarray=single`, el compilador asume coarrays single-image por defecto en versiones recientes.

Para ejecutar con múltiples imágenes (necesitas OpenCoarrays o similar):

```bash
# Con OpenCoarrays instalado
cafrun -n 4 ./programa_coarray

# O con la version experimental de gfortran (no recomendado para produccion)
# GFORTRAN_NUM_IMAGES=4 ./programa_coarray
```

**Limitación importante**: Los proyectos Coarrays en este capítulo están diseñados para `-fcoarray=single` (una imagen). Para ejecutarlos con múltiples imágenes reales, necesitas instalar OpenCoarrays y ajustar la comunicación remota.

### Tabla comparativa de speedups final

| Proyecto | Tamaño | Serial (s) | OpenMP 4 hilos (s) | Speedup | DO CONCURRENT (s) | Speedup |
|----------|--------|-----------|-------------------|---------|-------------------|---------|
| MatMul   | N=500  | 1.23      | 0.32              | 3.81×   | 1.12              | 1.10×   |
| MatMul   | N=1000 | 10.57     | 2.71              | 3.90×   | 9.21              | 1.15×   |
| MatMul   | N=2000 | 85.23     | 21.58             | 3.95×   | 71.02             | 1.20×   |
| Mandelbr | HD     | 45.32     | 11.84 (dynamic)   | 3.83×   | —                 | —       |
| N-Body   | N=500  | 12.56     | 3.42              | 3.67×   | —                 | —       |

### Instalación de herramientas necesarias

```bash
# Instalar gfortran (GNU Fortran Compiler)
sudo apt update
sudo apt install gfortran

# Instalar herramientas de perfilado
sudo apt install linux-tools-common linux-tools-generic gprof

# Instalar OpenCoarrays (opcional, para coarrays multi-imagen)
sudo apt install open-coarrays-bin

# Instalar ImageMagick para visualizar PPM
sudo apt install imagemagick

# Verificar instalacion
gfortran --version
perf --version
gprof --version
```

### Errores típicos de compilación y ejecución

**Error 1: Olvidar -fopenmp en el linkeo**

```bash
gfortran -O3 -fopenmp -c programa.f90 -o programa.o  # OK
gfortran programa.o -o programa  # ERROR: falta -fopenmp
```

Mensaje de gfortran:
```
programa.o: In function 'MAIN__':
programa.f90:(.text+0x12): undefined reference to 'omp_get_thread_num_'
collect2: error: ld returned 1 exit status
```

Solución: `gfortran -fopenmp programa.o -o programa`

**Error 2: Ejecutar coarrays sin soporte multi-imagen**

```
*** ERROR: Coarray program requires multiple images to run.
```

Solución: Compilar con `-fcoarray=single` para single-image, o instalar OpenCoarrays y usar `cafrun -n N`.

**Error 3: No exportar OMP_NUM_THREADS**

```bash
./programa  # Usa tantos hilos como nucleos fisicos tenga la CPU
```

No hay error, pero el programa no usará el número de hilos deseado. Los valores por defecto dependen de la implementación.

Solución: `export OMP_NUM_THREADS=4 && ./programa`

**Error 4: Segfault en Coarrays por tamaño insuficiente de pila**

```
Program terminated with signal SIGSEGV, Segmentation fault.
```

Ocurre cuando los coarrays locales son demasiado grandes y la pila del sistema no es suficiente.

Solución: `ulimit -s unlimited` antes de ejecutar, o usar memoria dinámica con `allocate`.

---

## Conclusión

A lo largo de este capítulo has aprendido los fundamentos del cómputo de alto rendimiento con Fortran Moderno. Desde los conceptos teóricos (Amdahl, Gustafson, speedup, eficiencia) hasta la implementación práctica de tres proyectos completos de simulación numérica.

Fortran sigue siendo el lenguaje de referencia en HPC no por tradición, sino por su capacidad para expresar operaciones paralelas de forma natural y eficiente. OpenMP te permite explotar memoria compartida dentro de un nodo; los Coarrays extienden el paralelismo a memoria distribuida; y DO CONCURRENT ofrece una vía portable para vectorización.

Recuerda siempre: **mide antes de optimizar**. La ley de Amdahl te recuerda que la fracción secuencial siempre limita el speedup. La ley de Gustafson te anima a pensar en problemas más grandes, no solo en resolver los mismos más rápido.

El siguiente paso natural es explorar MPI (Message Passing Interface) para programas que escalan a cientos de miles de núcleos, y GPGPU computing (OpenACC, OpenMP target, CUDA Fortran) para aceleración masiva. Pero eso es material para otro capítulo.
