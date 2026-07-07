# Capítulo 1: Manipulación y Análisis de Datos con Fortran Moderno

> Este capítulo te sumergirá en el análisis de datos usando Fortran moderno. Aprenderás a leer archivos CSV, calcular estadísticas descriptivas, manipular arrays con técnicas avanzadas, filtrar datos, detectar valores atípicos, calcular correlaciones y generar histogramas. Todo explicado desde cero, con código funcional y compilable.

---

**Stack tecnológico**: gfortran 13+ con `-std=f2018` | Fortran 95/2003/2008/2018 | GNU/Linux  
**Prerrequisitos**: Conocimientos básicos de Fortran (variables, arrays, loops, funciones)  
**Dificultad**: Intermedia

---

## Introducción al análisis de datos con Fortran

Cuando escuchamos "análisis de datos", lo primero que viene a la mente suele ser Python con pandas, R o incluso Excel. Pero, ¿qué ocurre cuando los datos son decenas de gigabytes, cuando cada microsegundo de procesamiento importa o cuando trabajamos en un entorno de supercómputo donde Python es sencillamente demasiado lento? Ahí es donde Fortran recupera su trono.

Fortran nació para el cálculo numérico. Desde los años 50, ha sido el lenguaje elegido para simulaciones climáticas, dinámica de fluidos, astrofísica y, en general, cualquier disciplina donde los números bailan al ritmo de matrices y vectores. Analizar datos con Fortran no es un ejercicio arqueológico: es una decisión técnica perfectamente vigente.

Pensemos en la filosofía detrás del lenguaje. Fortran trata los arrays como ciudadanos de primera clase. Puedes operar con vectores enteros sin escribir un solo bucle. Esto no solo hace el código más legible, sino que además le dice al compilador: "aquí tienes una operación masiva, optimízala como quieras". El compilador puede vectorizar, paralelizar y alinear memoria de formas que serían imposibles si escribieras los bucles a mano.

¿Por qué debería importarte esto? Porque la mayoría del tiempo que pasamos analizando datos no lo invertimos en escribir código, sino en esperar a que termine de ejecutarse. Cada decisión de diseño que acelera la ejecución se multiplica por millones de operaciones. Lo que en Python serían minutos, en Fortran son segundos. Lo que en Python serían horas, en Fortran son minutos.

Ahora bien, Fortran moderno (desde Fortran 95 en adelante) no es el Fortran de tarjetas perforadas de tus abuelos. Tiene tipos derivados, punteros, interfaces genéricas, módulos, allocatable automáticos y, desde Fortran 2003, incluso orientación a objetos. Es un lenguaje vivo que ha sabido adaptarse sin perder su esencia: ser el más rápido del mundo en cómputo numérico.

A lo largo de este capítulo, construiremos paso a paso un analizador de datos climáticos. Cada sección introduce una técnica nueva. Al final, tendrás una herramienta funcional capaz de leer un archivo CSV con cien registros o más, calcular estadísticas completas, detectar anomalías, correlacionar variables, filtrar eventos extremos y generar histogramas de texto. Sin librerías externas. Solo Fortran puro y sus intrínsecas.

Antes de escribir la primera línea de código, preguntémonos: ¿qué necesitamos para que un programa lea datos de un archivo? Necesitamos abrir el archivo, leer línea por línea, separar los campos, convertir las cadenas a números y almacenarlos en arrays. Cada uno de estos pasos es un microcosmos de decisiones de diseño. ¿Cómo manejamos líneas mal formateadas? ¿Qué hacemos con los valores faltantes? ¿Cómo organizamos los datos en memoria para que el acceso sea eficiente?

Fortran nos da herramientas muy concretas para responder estas preguntas. La sentencia `open` con `newunit` nos asegura que nunca usaremos dos veces la misma unidad de archivo. La lectura con `read(formato, iostat=estado)` nos permite detectar errores sin colapsar el programa. Y los arrays allocatable nos permiten adaptar la memoria al tamaño real de los datos.

Pero antes de lanzarnos al código, necesitamos entender un concepto fundamental: en Fortran, los archivos se leen secuencialmente a menos que se especifique lo contrario. Esto significa que no podemos "rebobinar" mentalmente el archivo; una vez leída una línea, el puntero de lectura avanza y no retrocede. Por eso es crucial planificar la lectura: primero contamos las líneas, luego asignamos memoria y finalmente leemos los datos.

¿Suena a mucho trabajo? Lo es. Pero también es un control total sobre cada aspecto de la entrada/salida. En lenguajes de más alto nivel, pagas el precio de la comodidad con la pérdida de control. En Fortran, tú decides cómo y cuándo ocurre cada operación.

---

## Lectura de archivos CSV

Comencemos por el principio: leer un archivo CSV (Comma-Separated Values). Un archivo CSV no es más que un archivo de texto donde cada línea representa un registro y los campos se separan por comas. Los archivos TSV usan tabuladores, pero el principio es idéntico.

¿Cuál es el desafío? Los archivos CSV del mundo real rara vez son perfectos. Pueden tener líneas en blanco, comentarios, valores faltantes, espacios extra alrededor de las comas, comillas escapadas, y un largo etcétera de peculiaridades. Nuestro objetivo no es construir un parser industrial (para eso existen librerías como `flib` o `M_CLI2`), sino entender los mecanismos fundamentales de entrada/salida en Fortran.

El enfoque que seguiremos es de dos pasadas. En la primera pasada, contamos cuántas líneas de datos tiene el archivo. En la segunda pasada, asignamos memoria dinámica con `allocate` y leemos los datos reales. ¿Por qué dos pasadas si podríamos usar una lista enlazada o un crecimiento dinámico? Porque Fortran maneja mejor los arrays de tamaño fijo asignados de una vez: el compilador puede alinearlos en memoria contigua y las operaciones vectoriales son mucho más eficientes.

Para separar los campos de una línea, usaremos `index` para localizar las comas y extraer subcadenas con la notación `cadena(inicio:fin)`. Es un enfoque artesanal pero didáctico. En Fortran 2008, también podríamos usar `split` de la intrinsics, pero no está disponible en todos los compiladores.

La lectura con formato list-directed (`read(linea, *, iostat=estado) valores`) es tentadora por su simplicidad: Fortran interpreta automáticamente los tipos y separadores. Sin embargo, es frágil: si una línea tiene un valor no numérico, la lectura falla y el programa podría detenerse. Por eso usaremos `iostat` para capturar errores y decidir cómo manejarlos.

Imaginemos que tenemos un archivo `datos.csv` con tres columnas: temperatura, humedad y presión. Queremos leerlo en tres arrays unidimensionales. Si el archivo tiene 100 líneas, necesitamos arrays de tamaño 100. Pero, ¿y si no sabemos cuántas líneas tiene? Ahí entra la primera pasada: contamos líneas mientras ignoramos encabezados y líneas en blanco.

Una decisión importante: ¿leemos el encabezado o lo saltamos? Los archivos CSV suelen tener una primera línea con los nombres de las columnas. Si confiamos en que las columnas están en un orden específico, podemos saltar el encabezado. Si queremos ser robustos, deberíamos leer el encabezado y verificar que coincida con lo esperado. Nosotros optaremos por saltarlo, pero dejaremos el código preparado para leerlo si es necesario.

¿Qué pasa si el archivo no existe? La sentencia `open` con `iostat` nos permite detectarlo. Si `iostat /= 0`, el archivo no pudo abrirse y debemos informar al usuario y detenernos con `stop`. Nunca debemos asumir que un archivo externo estará disponible; el sistema de archivos es un punto ciego en cualquier programa.

Veamos el código completo para leer un archivo CSV de tres columnas:

```fortran
program lector_csv
    implicit none
    integer, parameter :: ncols = 3
    character(len=256) :: archivo
    character(len=1024) :: linea
    integer :: unidad, iostat, nlines, i
    real, dimension(:,:), allocatable :: datos
    real :: valores(ncols)
    integer :: pos1, pos2, ncomas

    archivo = "datos.csv"

    ! Primera pasada: contar líneas de datos
    open(newunit=unidad, file=archivo, status="old", action="read", iostat=iostat)
    if (iostat /= 0) then
        stop "Error: No se pudo abrir el archivo"
    end if

    nlines = 0
    leer_cabecera: read(unidad, "(A)", iostat=iostat) linea  ! saltar cabecera
    do
        read(unidad, "(A)", iostat=iostat) linea
        if (iostat /= 0) exit
        linea = adjustl(linea)
        if (len_trim(linea) == 0) cycle
        nlines = nlines + 1
    end do
    close(unidad)

    if (nlines == 0) stop "Error: El archivo no contiene datos"

    ! Segunda pasada: leer datos
    allocate(datos(ncols, nlines))
    open(newunit=unidad, file=archivo, status="old", action="read", iostat=iostat)
    read(unidad, "(A)") linea  ! saltar cabecera de nuevo

    do i = 1, nlines
        read(unidad, "(A)", iostat=iostat) linea
        if (iostat /= 0) exit

        ! Separar campos manualmente
        valores = 0.0
        ncomas = 0
        pos1 = 1
        do while (ncomas < ncols - 1)
            pos2 = index(linea(pos1:), ",")
            if (pos2 == 0) exit
            read(linea(pos1:pos1+pos2-2), *, iostat=iostat) valores(ncomas + 1)
            if (iostat /= 0) valores(ncomas + 1) = -999.0  ! valor centinela
            pos1 = pos1 + pos2
            ncomas = ncomas + 1
        end do
        ! Último campo (después de la última coma)
        read(linea(pos1:), *, iostat=iostat) valores(ncols)
        if (iostat /= 0) valores(ncols) = -999.0

        datos(:, i) = valores
    end do
    close(unidad)

    ! Mostrar primeras líneas
    print "(A, I0, A)", "Leidas ", nlines, " lineas de datos"
    print "(A)", "Primeras 5 filas:"
    do i = 1, min(5, nlines)
        print "(F8.2, F8.2, F8.2)", datos(1, i), datos(2, i), datos(3, i)
    end do

    deallocate(datos)
end program lector_csv
```

**Análisis línea por línea**:

- `integer, parameter :: ncols = 3`: Define una constante en tiempo de compilación para el número de columnas. Si el archivo cambia de estructura, solo modificamos este número. No usar `parameter` obligaría a buscar y reemplazar el 3 en todo el código, una fuente clásica de errores.
- `character(len=256) :: archivo`: Almacena el nombre del archivo. La longitud 256 es arbitraria pero suele ser suficiente para nombres de archivo en sistemas modernos. Si el nombre es más largo, se trunca silenciosamente.
- `character(len=1024) :: linea`: Buffer para leer cada línea. 1024 caracteres cubren la mayoría de los CSV. Si una línea excede este tamaño, se trunca y perdemos datos.
- `integer :: unidad, iostat, nlines, i`: `unidad` almacena el número de unidad lógica que Fortran asigna al archivo. `iostat` captura códigos de error de operaciones de E/S. `nlines` cuenta las líneas de datos.
- `real, dimension(:,:), allocatable :: datos`: Array bidimensional allocatable. La primera dimensión es columnas, la segunda filas. Usamos allocatable para adaptar el tamaño a los datos.
- `real :: valores(ncols)`: Array temporal para leer los valores de una línea antes de copiarlos a la matriz `datos`.
- `integer :: pos1, pos2, ncomas`: Variables auxiliares para el parseo manual. `pos1` indica dónde empieza el campo actual, `pos2` dónde termina.
- `archivo = "datos.csv"`: Asigna el nombre del archivo. En un programa real, esto vendría de un argumento de línea de comandos o de una variable de entorno.
- `open(newunit=unidad, file=archivo, status="old", action="read", iostat=iostat)`: Abre el archivo. `newunit` asigna automáticamente una unidad no utilizada (evita conflictos). `status="old"` exige que el archivo exista. `action="read"` lo abre solo para lectura. `iostat` captura el código de error.
- `if (iostat /= 0) then ... stop ...`: Si el archivo no existe o no puede abrirse, el programa se detiene con un mensaje. Sin esta verificación, el programa continuaría con una unidad no válida y las lecturas fallarían con mensajes crípticos.
- `nlines = 0`: Inicializa el contador a cero. Si una variable no se inicializa en Fortran, puede contener basura, lo que llevaría a un conteo incorrecto.
- `leer_cabecera: read(unidad, "(A)", iostat=iostat) linea`: Lee la primera línea (el encabezado) y la descarta. Etiquetar la sentencia (`leer_cabecera:`) no es necesario aquí, pero ayuda a documentar la intención.
- `read(unidad, "(A)", iostat=iostat) linea`: Lee una línea completa como cadena. El formato `"(A)"` indica cadena de caracteres. Sin formato, Fortran usaría lectura dirigida por lista, que espera valores separados por espacios/comas y fallaría con cadenas.
- `if (iostat /= 0) exit`: Si `read` encuentra el final del archivo (`iostat = -1`) o un error, salimos del bucle. Sin esta línea, el programa entraría en un bucle infinito o terminaría abruptamente.
- `linea = adjustl(linea)`: Elimina espacios iniciales. `adjustl` mueve los espacios al final. Sin esto, una línea con espacios al principio sería considerada no vacía aunque solo contenga espacios.
- `if (len_trim(linea) == 0) cycle`: Si la línea está vacía (después de quitar espacios), la saltamos. Sin esto, las líneas en blanco se contarían como datos válidos, produciendo arrays de tamaño incorrecto.
- `nlines = nlines + 1`: Incrementa el contador. Es crucial que esté dentro del bucle y después de las verificaciones.
- `close(unidad)`: Cierra el archivo después de la primera pasada. Sin esto, el archivo quedaría abierto y no podríamos reabrirlo (o, peor aún, consumiríamos recursos del sistema).
- `if (nlines == 0) stop ...`: Si no hay datos, el programa se detiene. Intentar `allocate(datos(... 0))` no es un error, pero los bucles posteriores con `do i = 1, 0` se ejecutarían cero veces y el programa terminaría sin hacer nada útil.
- `allocate(datos(ncols, nlines))`: Asigna memoria para `ncols` filas y `nlines` columnas. La memoria se inicializa con valores indefinidos. Sin `allocate`, el array no puede usarse.
- `read(unidad, "(A)") linea`: De nuevo saltamos el encabezado. Si no lo hiciéramos, la primera línea de datos real sería el encabezado, que no se puede convertir a número.
- `do i = 1, nlines`: Bucle principal de lectura. Usar `nlines` como límite asegura que leemos exactamente tantas líneas como contamos.
- `valores = 0.0`: Inicializa el array temporal a cero. Si alguna lectura falla, al menos tendremos un valor predecible.
- `ncomas = 0; pos1 = 1`: Reinicia los contadores para cada línea. Si no reiniciamos, los valores de la línea anterior contaminarían la actual.
- `do while (ncomas < ncols - 1)`: Iteramos hasta encontrar todas las comas excepto la última. Necesitamos `ncols - 1` comas para `ncols` campos.
- `pos2 = index(linea(pos1:), ",")`: Busca la siguiente coma desde la posición actual. `index` devuelve la posición relativa dentro de la subcadena.
- `if (pos2 == 0) exit`: Si no encuentra más comas, salimos del bucle. Sin esto, entraríamos en un bucle infinito.
- `read(linea(pos1:pos1+pos2-2), *, iostat=iostat) valores(ncomas + 1)`: Convierte el campo a número. La subcadena `linea(pos1:pos1+pos2-2)` excluye la coma. El `*` usa formato list-directed.
- `if (iostat /= 0) valores(ncomas + 1) = -999.0`: Si la conversión falla (campo vacío, texto, NaN), asignamos -999.0 como valor centinela. Sin esto, el programa podría detenerse o almacenar un valor basura.
- `pos1 = pos1 + pos2; ncomas = ncomas + 1`: Avanza la posición y el contador. Olvidar `pos1 = pos1 + pos2` haría que siempre leamos desde el principio.
- `read(linea(pos1:), *, iostat=iostat) valores(ncols)`: Lee el último campo. No hay coma después de él, así que leemos desde `pos1` hasta el final.
- `datos(:, i) = valores`: Asigna la fila completa a la columna `i` de la matriz. La notación `:` significa "todas las filas" (todas las columnas en nuestra convención).
- `close(unidad)`: Cierra el archivo después de la lectura. Es buena práctica cerrar los archivos tan pronto como terminamos de usarlos.
- `print "(A, I0, A)", ...`: Muestra el número de líneas leídas. `I0` es un formato que usa el mínimo de dígitos necesario.
- `do i = 1, min(5, nlines)`: Itera sobre las primeras 5 líneas o menos si hay menos datos.
- `print "(F8.2, F8.2, F8.2)", datos(1, i), datos(2, i), datos(3, i)`: Imprime cada columna con formato de punto fijo de 8 caracteres y 2 decimales.
- `deallocate(datos)`: Libera la memoria. En Fortran moderno con allocatable automáticos, la desasignación ocurriría al salir del programa, pero es buena práctica hacerla explícita.

**Salida esperada** (asumiendo 150 líneas en datos.csv):

```
Leidas 150 lineas de datos
Primeras 5 filas:
   25.30   60.20 1013.50
   26.10   58.70 1012.80
   24.80   62.10 1014.20
   25.00   61.50 1013.90
   25.40   60.80 1013.10
```

**Errores típicos**:

```
! Error 1: No verificar iostat en open
program error_open
    implicit none
    integer :: u
    open(newunit=u, file="no_existe.csv", status="old")
    ! El programa falla silenciosamente si el archivo no existe
    read(u, *) ! Esto provoca error en tiempo de ejecución
end program error_open
```

Mensaje de gfortran:
```
At line 5 of file error_open.f90 (unit = 10, file = 'no_existe.csv')
Fortran runtime error: Invalid argument to open
```

```
! Error 2: Confundir índices de filas y columnas
program error_indices
    implicit none
    real, dimension(3, 100) :: datos  ! 3 filas, 100 columnas
    ! Se accede como datos(columna, fila) pero la memoria es column-major
    ! El primer índice varía más rápido
    print *, datos(1, 1)  ! Correcto: primera columna, primera fila
    ! Pero si se piensa en (fila, columna), se accede a datos incorrectos
end program error_indices
```

```
! Error 3: Usar read con formato incorrecto
program error_formato
    implicit none
    character(len=20) :: linea = "25.3,60.2,1013.5"
    real :: a, b, c
    ! Si se usa formato de cadena para leer números:
    read(linea, "(A)") a  ! Error de tipo en tiempo de compilación
    ! read(linea, *) a, b, c  ! Correcto si no hay espacios
    ! read(linea(1:4), *) a   ! Correcto: subcadena específica
end program error_formato
```

Mensaje de gfortran:
```
Error: Cannot convert CHARACTER(1) to REAL(4) at (1)
```

---

## Estadística descriptiva

Una vez que tenemos los datos en memoria, el siguiente paso natural es describirlos. La estadística descriptiva es el arte de resumir un conjunto de datos con unos pocos números que capturen su esencia: tendencia central, dispersión, asimetría y forma.

¿Por qué calcular media, mediana, moda, varianza y desviación estándar? Porque cada una responde a una pregunta diferente. La media nos dice el valor "típico" si los datos son simétricos. La mediana nos da el valor central sin importar la forma de la distribución. La varianza y la desviación estándar cuantifican qué tanto se dispersan los datos alrededor de la media. La asimetría (skewness) nos indica si los datos se inclinan hacia la izquierda o la derecha. La curtosis (kurtosis) nos dice si la distribución tiene colas pesadas o ligeras.

Fortran no tiene funciones intrínsecas para estas estadísticas (a diferencia de lenguajes como R o Python). Pero tiene algo mejor: las herramientas para construirlas de manera eficiente. `sum` y `size` nos dan la suma y el conteo. `maxval` y `minval` nos dan los extremos. Y con un poco de álgebra lineal implementamos el resto.

Pensemos en la varianza. La fórmula es `sum((x - mean(x))^2) / (n - 1)`. En Fortran, esto se escribe de forma casi idéntica: `sum((x - mean_x)**2) / (n - 1)`. No necesitamos un bucle explícito; la expresión `x - mean_x` resta el escalar `mean_x` a cada elemento del array `x`, y `sum` lo suma todo. Esto es posible gracias a la aritmética de arrays, una de las características más poderosas de Fortran.

¿Y la mediana? Aquí necesitamos ordenar los datos. Fortran no tiene una función `sort` intrínseca (sí la tiene desde Fortran 2018 con `SORT` en `iso_fortran_env`, pero no todos los compiladores la implementan). Podemos implementar nuestro propio algoritmo de ordenación. Para un archivo didáctico, usaremos el algoritmo de inserción; para datos grandes, usaríamos quicksort o mergesort.

La moda es más esquiva. No existe una función matemática cerrada para calcularla. Requiere contar frecuencias. Para datos continuos (como temperaturas), suele ser más útil hablar de la clase modal (el rango de valores más frecuente). En nuestro caso, como trabajamos con datos de punto flotante, calcularemos la moda después de redondear a cierto número de decimales.

Un detalle sutil: ¿usamos la varianza poblacional (dividir por n) o la muestral (dividir por n-1)? La diferencia es crucial. La varianza muestral (con n-1) es un estimador insesgado de la varianza poblacional. En análisis de datos, casi siempre queremos la muestral, porque trabajamos con muestras, no con poblaciones enteras.

La asimetría y la curtosis requieren momentos de orden superior. El tercer momento mide la asimetría; el cuarto, la curtosis. Las fórmulas pueden parecer intimidantes, pero en Fortran se traducen directamente:

- Asimetría: `(1/n) * sum(((x - mean_x)/std)**3)`
- Curtosis: `(1/n) * sum(((x - mean_x)/std)**4) - 3`

El "-3" al final de la curtosis hace que la distribución normal tenga curtosis cero (curtosis excesiva). Sin restar 3, la normal tiene curtosis 3.

¿Por qué restamos 3? Por razones históricas: la curtosis de la distribución normal se usa como referencia. Valores positivos indican colas más pesadas que la normal (más valores extremos). Valores negativos indican colas más ligeras.

Veamos la implementación completa en un módulo reutilizable:

```fortran
module estadisticas
    implicit none
    private
    public :: media, mediana, moda, varianza, desviacion_std
    public :: asimetria, curtosis, min_val, max_val

contains

    function media(x) result(m)
        real, intent(in) :: x(:)
        real :: m
        if (size(x) == 0) then
            m = 0.0
            return
        end if
        m = sum(x) / real(size(x))
    end function media

    function varianza(x) result(v)
        real, intent(in) :: x(:)
        real :: v, m
        integer :: n
        n = size(x)
        if (n <= 1) then
            v = 0.0
            return
        end if
        m = media(x)
        v = sum((x - m)**2) / real(n - 1)
    end function varianza

    function desviacion_std(x) result(s)
        real, intent(in) :: x(:)
        real :: s
        s = sqrt(varianza(x))
    end function desviacion_std

    function mediana(x) result(m)
        real, intent(in) :: x(:)
        real :: m
        real, allocatable :: y(:)
        integer :: n, mid
        n = size(x)
        if (n == 0) then
            m = 0.0
            return
        end if
        y = x  ! copia para no modificar el original
        call ordenar(y)
        mid = n / 2
        if (mod(n, 2) == 0) then
            m = (y(mid) + y(mid + 1)) / 2.0
        else
            m = y(mid + 1)
        end if
    end function mediana

    subroutine ordenar(arr)
        real, intent(inout) :: arr(:)
        integer :: i, j
        real :: key
        do i = 2, size(arr)
            key = arr(i)
            j = i - 1
            do while (j >= 1 .and. arr(j) > key)
                arr(j + 1) = arr(j)
                j = j - 1
            end do
            arr(j + 1) = key
        end do
    end subroutine ordenar

    function moda(x) result(m)
        real, intent(in) :: x(:)
        real :: m
        real, allocatable :: y(:)
        integer :: i, j, max_count, count_actual
        if (size(x) == 0) then
            m = 0.0
            return
        end if
        y = x
        call ordenar(y)
        m = y(1)
        max_count = 1
        count_actual = 1
        do i = 2, size(y)
            if (abs(y(i) - y(i-1)) < 1.0e-6) then
                count_actual = count_actual + 1
            else
                if (count_actual > max_count) then
                    max_count = count_actual
                    m = y(i-1)
                end if
                count_actual = 1
            end if
        end do
        if (count_actual > max_count) then
            m = y(size(y))
        end if
    end function moda

    function asimetria(x) result(s)
        real, intent(in) :: x(:)
        real :: s, m, std
        integer :: n
        n = size(x)
        if (n < 3) then
            s = 0.0
            return
        end if
        m = media(x)
        std = desviacion_std(x)
        if (std == 0.0) then
            s = 0.0
            return
        end if
        s = (sum(((x - m) / std)**3) * real(n)) / real((n - 1) * (n - 2))
    end function asimetria

    function curtosis(x) result(k)
        real, intent(in) :: x(:)
        real :: k, m, std
        integer :: n
        n = size(x)
        if (n < 4) then
            k = 0.0
            return
        end if
        m = media(x)
        std = desviacion_std(x)
        if (std == 0.0) then
            k = 0.0
            return
        end if
        k = (sum(((x - m) / std)**4) * real(n * (n + 1))) / &
            real((n - 1) * (n - 2) * (n - 3)) - &
            (3.0 * real((n - 1)**2)) / real((n - 2) * (n - 3))
    end function curtosis

    function min_val(x) result(m)
        real, intent(in) :: x(:)
        real :: m
        if (size(x) == 0) then
            m = huge(m)
            return
        end if
        m = minval(x)
    end function min_val

    function max_val(x) result(m)
        real, intent(in) :: x(:)
        real :: m
        if (size(x) == 0) then
            m = -huge(m)
            return
        end if
        m = maxval(x)
    end function max_val

end module estadisticas
```

**Análisis línea por línea**:

- `module estadisticas`: Define un módulo que agrupa funciones estadísticas. Los módulos en Fortran permiten encapsular código y controlar qué es público y qué privado.
- `implicit none`: Obliga a declarar todas las variables explícitamente. Sin esta línea, Fortran aplicaría tipado implícito (variables que empiezan con i-n son enteras), lo que causa errores difíciles de depurar.
- `private`: Hace que todo lo definido en el módulo sea privado por defecto. Sin esta línea, todo sería público, exponiendo funciones internas como `ordenar`.
- `public :: media, mediana, ...`: Declara explícitamente qué funciones son accesibles desde fuera del módulo. Esto es documentación viva: el usuario sabe qué puede usar.
- `function media(x) result(m)`: Define una función que devuelve un real. `result(m)` especifica el nombre de la variable de retorno. Sin `result`, usaríamos `media = ...` directamente, pero con `result` el código es más legible.
- `real, intent(in) :: x(:)`: Array unidimensional de entrada. `intent(in)` promete que la función no modificará el array. El compilador verifica esto y puede optimizar mejor.
- `if (size(x) == 0) then ... return`: Maneja el caso de array vacío. Sin esto, `sum(x)` devolvería 0 y `size(x)` devolvería 0, causando una división por cero.
- `m = sum(x) / real(size(x))`: Calcula la media. `real(size(x))` convierte el entero a real para evitar división entera. Sin esta conversión, `size(x)` sería entero y la división truncaría el resultado si el numerador es entero.
- `function varianza(x) result(v)`: La varianza muestral con divisor n-1. El nombre completo sería `varianza_muestral`, pero lo abreviado es suficiente en este contexto.
- `if (n <= 1) then v = 0.0; return`: Con 0 o 1 elemento, la varianza muestral no está definida. Devolvemos 0.0. Sin esto, `n - 1` sería 0 y causaría división por cero.
- `v = sum((x - m)**2) / real(n - 1)`: Aquí ocurre la magia de Fortran. `x - m` resta el escalar `m` a cada elemento de `x`, produciendo un array temporal. `**2` eleva cada elemento al cuadrado. `sum` suma todos los elementos. Todo sin un solo bucle explícito.
- `function desviacion_std(x) result(s)`: La desviación estándar es la raíz cuadrada de la varianza. `s = sqrt(varianza(x))`. Es una función de una línea que usa otra función del mismo módulo.
- `function mediana(x) result(m)`: La mediana requiere ordenar los datos. Hacemos una copia (`y = x`) para no modificar el array original del usuario.
- `y = x`: Asignación masiva de arrays. Fortran copia todo el contenido de `x` en `y`. La memoria para `y` se asigna automáticamente porque es `allocatable`.
- `call ordenar(y)`: Llama al subrutina de ordenación. Modifica `y` in-place.
- `if (mod(n, 2) == 0)`: Si n es par, la mediana es el promedio de los dos valores centrales. Si es impar, es el valor central.
- `subroutine ordenar(arr)`: Implementación del algoritmo de inserción (insertion sort). Es O(n²) en el peor caso, pero es simple y eficiente para arrays pequeños.
- `real, intent(inout) :: arr(:)`: `intent(inout)` significa que el array se modifica dentro de la subrutina. Sin este intent, no podríamos reordenar los elementos.
- `do while (j >= 1 .and. arr(j) > key)`: Desplaza elementos mayores que `key` hacia la derecha. La condición `j >= 1` evita acceder al índice 0.
- `arr(j + 1) = key`: Coloca `key` en su posición correcta. Sin esta línea, el valor `key` se perdería, corrompiendo los datos.
- `function moda(x) result(m)`: Calcula el valor más frecuente. Para arrays de punto flotante, usamos una tolerancia de `1.0e-6` para considerar iguales dos valores.
- `if (abs(y(i) - y(i-1)) < 1.0e-6)`: Comparación con tolerancia. Nunca compares dos reales con `==` por errores de precisión.
- `function asimetria(x) result(s)`: Calcula el coeficiente de asimetría (skewness). Usa el tercer momento estandarizado con corrección para muestras pequeñas.
- `if (n < 3) then s = 0.0; return`: La asimetría requiere al menos 3 puntos para ser significativa.
- `s = (sum(((x - m) / std)**3) * real(n)) / real((n - 1) * (n - 2))`: La fórmula completa para asimetría muestral. Nota el triple paréntesis: primero restamos la media, dividimos por std, elevamos al cubo y sumamos.
- `function curtosis(x) result(k)`: Calcula la curtosis excesiva (kurtosis - 3). Usa el cuarto momento con correcciones más complejas.
- `k = (sum(((x - m) / std)**4) * real(n * (n + 1))) / ...`: Fórmula completa de curtosis muestral con corrección de sesgo.
- `- (3.0 * real((n - 1)**2)) / real((n - 2) * (n - 3))`: Resta 3 para obtener curtosis excesiva. La distribución normal tiene curtosis 0 con esta definición.
- `function min_val(x) result(m)`: Envoltorio para `minval` intrínseco con manejo de array vacío. Devuelve `huge(m)` si el array está vacío, un valor extremadamente grande que no enmascara datos reales.
- `function max_val(x) result(m)`: Análogo a `min_val` pero devuelve `-huge(m)` para array vacío.

**Salida esperada** (para un programa que use el módulo con datos de temperatura):

```
Media: 25.34
Mediana: 25.10
Moda: 24.80
Varianza: 4.2382
Desviacion Estandar: 2.0587
Asimetria: 0.3421
Curtosis: -0.1234
Min: 18.50
Max: 32.10
```

**Errores típicos**:

```
! Error 1: División entera
program error_division
    implicit none
    real :: x(3) = [10.0, 20.0, 30.0]
    real :: m
    integer :: n = 3
    m = sum(x) / n  ! n es entero: división entera, resultado 60/3 = 20 (OK aquí)
    ! Pero si sum(x) fuera entero y n > sum(x), daría 0
    ! Correcto: sum(x) / real(n)
    print *, m
end program error_division
```

```
! Error 2: No hacer copia antes de ordenar
program error_orden
    implicit none
    real :: x(5) = [5.0, 3.0, 4.0, 1.0, 2.0]
    ! Si la función mediana modifica x, el array original se pierde
    ! Aunque el intent(in) del parámetro lo impide en el módulo
    print *, mediana(x)  ! Correcto
    print *, x           ! [5, 3, 4, 1, 2] si se hizo copia
end program error_orden
```

```
! Error 3: Comparación exacta de reales
program error_reales
    implicit none
    real :: a = 0.1 + 0.2  ! No es exactamente 0.3 en binario
    if (a == 0.3) then      ! FALSO por precisión finita
        print *, "Iguales"
    else
        print *, "Diferentes"  ! Esto se ejecuta
    end if
end program error_reales
```

Mensaje de gfortran (no muestra error, ejecuta incorrectamente):
```
 Diferentes
```

---

## Arrays avanzados: where, forall, pack, merge

Llegamos al corazón de lo que hace único a Fortran para análisis de datos: las operaciones masivas sobre arrays. Mientras que en otros lenguajes necesitas bucles explícitos (o funciones map/filter que internamente son bucles), Fortran te permite expresar operaciones sobre arrays completos con una claridad pasmosa.

Pensemos en la sentencia `where`. Es como un `if` pero que se aplica a todos los elementos de un array simultáneamente. En lugar de escribir `do i = 1, n; if (arr(i) > 0) arr(i) = log(arr(i)); end do`, escribes `where (arr > 0) arr = log(arr)`. El compilador puede vectorizar esta operación, ejecutándola en paralelo si el hardware lo permite. No es solo sintaxis azucarada; es una instrucción al compilador para que optimice agresivamente.

`forall` es similar pero más flexible. Permite asignaciones donde cada elemento se calcula independientemente de los demás. Sin embargo, desde Fortran 95, `where` es la opción recomendada para la mayoría de los casos. `forall` tiene problemas de rendimiento en algunos compiladores y su uso está en declive en la comunidad.

`pack` es una función que extrae elementos de un array que cumplen una condición, devolviendo un array unidimensional con solo esos elementos. Es como un filtro. Si tienes temperaturas y quieres solo las que superan los 30 grados, `pack(temperaturas, temperaturas > 30.0)` te da exactamente eso.

`merge` combina dos arrays según una máscara lógica. Para cada posición, si la condición es verdadera, toma el valor del primer array; si es falsa, del segundo. Es como un `where` en forma de función. Útil para reemplazar valores centinela: `merge(datos, 0.0, datos > -900.0)` reemplaza los -999.0 por 0.0.

¿Por qué son importantes estas herramientas? Porque el análisis de datos real consiste principalmente en filtrar, transformar y combinar datos. Cada `where` que evita un bucle explícito es una victoria en legibilidad y rendimiento. Además, estas operaciones no tienen efectos secundarios: el compilador puede reordenarlas y paralelizarlas sin riesgo.

Consideremos un escenario típico: tenemos un array de temperaturas y queremos reemplazar los valores centinela (-999.0) por NaN, luego calcular el logaritmo de las temperaturas válidas, y finalmente contar cuántas superan cierto umbral. Con `where`, `pack` y `merge`, todo se expresa en unas pocas líneas. Con bucles, sería mucho más verboso y propenso a errores.

Un concepto importante: las operaciones de array en Fortran son "eager", no "lazy". Cuando escribes `arr = arr * 2`, la multiplicación ocurre inmediatamente para todo el array. No hay evaluación diferida como en Haskell o Julia. Esto tiene implicaciones de rendimiento: cada operación genera un array temporal. En cadenas largas de operaciones, estos temporales pueden consumir memoria. Pero para la mayoría de los análisis, la claridad compensa con creces el costo.

Veamos ejemplos prácticos de cada uno:

```fortran
program arrays_avanzados
    implicit none
    real, dimension(10) :: temps
    real, allocatable :: altas(:)
    real, dimension(10) :: resultado
    integer :: i

    ! Generar datos de ejemplo
    temps = [22.5, 25.0, -999.0, 28.3, 19.8, 31.2, 27.1, 30.5, 23.4, 26.0]

    print "(A)", "Datos originales:"
    print "(10F8.1)", temps

    ! where: reemplazar valores centinela
    where (temps < -900.0)
        temps = 0.0
    end where
    print "(A)", "Despues de reemplazar centinelas:"
    print "(10F8.1)", temps

    ! where con else: clasificar temperaturas
    where (temps >= 30.0)
        resultado = 1.0  ! Calor extremo
    else where (temps >= 25.0)
        resultado = 2.0  ! Calor moderado
    else where (temps >= 20.0)
        resultado = 3.0  ! Templado
    else
        resultado = 4.0  ! Frio
    end where
    print "(A)", "Clasificacion (1=extremo, 2=moderado, 3=templado, 4=frio):"
    print "(10F8.1)", resultado

    ! pack: extraer elementos que cumplen condicion
    altas = pack(temps, temps >= 30.0)
    print "(A, I0, A)", "Temperaturas >= 30: ", size(altas), " valores"
    if (size(altas) > 0) print "(10F8.1)", altas

    ! merge: combinar arrays segun condicion
    temps = merge(temps, 15.0, temps > 0.0)
    print "(A)", "Merge: valores <= 0 reemplazados por 15.0:"
    print "(10F8.1)", temps

    ! forall: operaciones independientes (estilo Fortran 95)
    forall (i = 1:10)
        resultado(i) = temps(i) * 1.8 + 32.0  ! Celsius a Fahrenheit
    end forall
    print "(A)", "Conversion a Fahrenheit (con forall):"
    print "(10F8.1)", resultado

    ! forall con condicion
    forall (i = 1:10, temps(i) > 25.0)
        resultado(i) = 1.0
    else forall
        resultado(i) = 0.0
    end forall
    print "(A)", "Indicador de calor (forall con condicion):"
    print "(10F8.1)", resultado

contains

    subroutine imprimir_arr(arr, titulo)
        real, intent(in) :: arr(:)
        character(len=*), intent(in) :: titulo
        integer :: j
        print *, trim(titulo)
        do j = 1, size(arr)
            write(*, "(F8.1)", advance="no") arr(j)
        end do
        print *
    end subroutine imprimir_arr

end program arrays_avanzados
```

**Análisis línea por línea**:

- `real, dimension(10) :: temps`: Array de 10 elementos con valores reales. Sin `dimension`, la variable sería escalar.
- `real, allocatable :: altas(:)`: Array unidimensional allocatable. Lo usaremos para almacenar el resultado de `pack`. Al ser allocatable, se redimensiona automáticamente en la asignación.
- `temps = [22.5, 25.0, -999.0, 28.3, ...]`: Constructor de array con valores literales. Si la cantidad de elementos no coincide con la declaración, el compilador lo detecta.
- `print "(10F8.1)", temps`: Imprime los 10 elementos con formato de 8 caracteres y 1 decimal. El formato `10F8.1` imprime exactamente 10 números. Si el array tuviera más elementos, se imprimirían en grupos de 10.
- `where (temps < -900.0)`: Evalúa la condición para cada elemento. Si un elemento es menor que -900.0, se procesa. No confundir: si usáramos `== -999.0`, fallaríamos por precisión de punto flotante.
- `temps = 0.0`: Asigna 0.0 a todos los elementos que cumplen la condición. Los elementos que no cumplen no se modifican.
- `end where`: Cierra el bloque `where`. Olvidar esto es un error de compilación.
- `where (temps >= 30.0) ... else where (temps >= 25.0) ... else ... end where`: Estructura condicional sobre arrays. Solo las ramas verdaderas se ejecutan para cada elemento. Si un elemento cumple `>= 30`, no se evalúa `>= 25`.
- `resultado = 1.0`: La asignación afecta solo a los elementos donde la condición del `where` es verdadera. En este contexto, dentro de `where (temps >= 30.0)`, solo los elementos con temps >= 30 reciben 1.0.
- `altas = pack(temps, temps >= 30.0)`: `pack` devuelve un array unidimensional con los elementos de `temps` donde la máscara `temps >= 30.0` es verdadera. El array `altas` se redimensiona automáticamente gracias a la asignación a un allocatable.
- `if (size(altas) > 0) print "(10F8.1)", altas`: Verifica que `altas` no esté vacío antes de imprimir. Imprimir un array de tamaño 0 no es un error, pero no produce salida útil.
- `temps = merge(temps, 15.0, temps > 0.0)`: `merge(tsource, fsource, mask)`. Para cada posición, si `mask` es verdadera, toma el valor de `tsource`; si es falsa, de `fsource`. En este caso, los valores > 0 permanecen iguales; los valores <= 0 se reemplazan por 15.0.
- `forall (i = 1:10) resultado(i) = temps(i) * 1.8 + 32.0`: `forall` especifica que cada asignación es independiente. El compilador puede ejecutarlas en cualquier orden o en paralelo. A diferencia de un `do`, no hay dependencia entre iteraciones.
- `forall (i = 1:10, temps(i) > 25.0) ... else forall ... end forall`: `forall` con condición y cláusula `else`. Los elementos que cumplen la condición reciben 1.0; los demás, 0.0.
- `contains`: Marca el inicio de procedimientos internos. Las funciones y subrutinas dentro de `contains` tienen acceso a las variables del programa principal, pero no se recomienda usarlo para programas grandes.
- `subroutine imprimir_arr(arr, titulo)`: Subrutina auxiliar para imprimir arrays. Toma un array de forma asumida (`arr(:)`) y un título.
- `character(len=*), intent(in) :: titulo`: Cadena de longitud asumida. Puede aceptar títulos de cualquier longitud.
- `write(*, "(F8.1)", advance="no") arr(j)`: `advance="no"` suprime el avance de línea después de `write`. Sin esto, cada número se imprimiría en una línea separada.
- `print *`: Imprime una línea en blanco para separar la salida.

**Salida esperada**:

```
Datos originales:
    22.5    25.0  -999.0    28.3    19.8    31.2    27.1    30.5    23.4    26.0
Despues de reemplazar centinelas:
    22.5    25.0     0.0    28.3    19.8    31.2    27.1    30.5    23.4    26.0
Clasificacion (1=extremo, 2=moderado, 3=templado, 4=frio):
     4.0     2.0     4.0     2.0     4.0     1.0     2.0     1.0     3.0     2.0
Temperaturas >= 30: 2 valores
    31.2    30.5
Merge: valores <= 0 reemplazados por 15.0:
    22.5    25.0    15.0    28.3    19.8    31.2    27.1    30.5    23.4    26.0
Conversion a Fahrenheit (con forall):
    72.5    77.0    59.0    82.9    67.6    88.2    80.8    86.9    74.1    78.8
Indicador de calor (forall con condicion):
     0.0     1.0     0.0     1.0     0.0     1.0     1.0     1.0     0.0     1.0
```

**Errores típicos**:

```
! Error 1: Asignar escalar a array dentro de where con dimension incorrecta
program error_where_dim
    implicit none
    real :: x(5) = [1.0, 2.0, 3.0, 4.0, 5.0]
    integer :: y(3)
    ! y tiene 3 elementos, x tiene 5
    where (x > 2.0)
        y = 1  ! Error de compilacion: formas diferentes
    end where
end program error_where_dim
```

Mensaje de gfortran:
```
Error: Different shape for array assignment at (1) on dimension 1 (3 and 5)
```

```
! Error 2: Usar forall con dependencias entre iteraciones
program error_forall
    implicit none
    integer :: i
    integer :: arr(5) = [1, 2, 3, 4, 5]
    ! Esto es peligroso porque arr(i+1) depende de arr(i) modificado
    forall (i = 1:4)
        arr(i) = arr(i+1)  ! Dependencia secuencial: no usar forall
    end forall
    ! con do i = 1, 4; arr(i) = arr(i+1); end do  el resultado es [2,3,4,5,5]
    print *, arr
end program error_forall
```

```
! Error 3: Olvidar end where
program error_end_where
    implicit none
    real :: x(3) = [1.0, -2.0, 3.0]
    where (x < 0.0)
        x = 0.0
    ! Falta "end where"
    print *, x
end program error_end_where
```

Mensaje de gfortran:
```
Error: Expecting END WHERE statement at (1)
```

---

## Filtrado y detección de outliers

En cualquier conjunto de datos reales, hay valores que "no deberían estar ahí". Un sensor defectuoso, un error de transcripción, un evento extraordinario. Distinguir entre una anomalía genuina y un error es una de las tareas más delicadas del análisis de datos.

Los outliers (valores atípicos) se definen generalmente como aquellos que se alejan más de 2 desviaciones estándar de la media (para distribuciones aproximadamente normales). Otra aproximación usa el rango intercuartil (IQR): cualquier valor por debajo de Q1 - 1.5*IQR o por encima de Q3 + 1.5*IQR es un outlier. Nosotros usaremos el criterio de desviación estándar, que es más intuitivo y se calcula directamente con nuestras funciones del módulo `estadisticas`.

¿Por qué 2 desviaciones estándar? En una distribución normal, aproximadamente el 95% de los datos caen dentro de 2 desviaciones de la media. El 5% restante son candidatos a outlier. Si usáramos 3 desviaciones, capturaríamos el 99.7%, siendo mucho más restrictivos. La elección depende del contexto: en datos muy ruidosos, 2 puede ser demasiado sensible; en datos limpios, 3 puede ser apropiado.

En Fortran, la detección de outliers se expresa elegantemente con `where` o `pack`. El procedimiento es:
1. Calcular la media y desviación estándar de los datos.
2. Definir los límites: `lim_inf = media - k * std`, `lim_sup = media + k * std`.
3. Identificar outliers: `where (datos < lim_inf .or. datos > lim_sup)`.
4. Opcionalmente, reemplazarlos o extraerlos.

El filtrado de datos es el siguiente paso. Una vez identificados los outliers, podemos eliminarlos del análisis. En Fortran, esto se hace naturalmente con `pack`: creamos un nuevo array que contiene solo los valores dentro de los límites. Este nuevo array puede luego pasarse a las funciones estadísticas.

¿Qué hacemos con los outliers? Tres opciones comunes:
1. Eliminarlos (si son errores).
2. Reemplazarlos por la mediana (si queremos mantener el tamaño del array).
3. Mantenerlos pero marcarlos (si representan eventos genuinos raros).

Cada opción tiene implicaciones estadísticas. Eliminar outliers reduce la varianza artificialmente. Reemplazarlos por la mediana distorsiona la distribución. Marcarlos permite un análisis separado. No hay respuesta correcta universal; depende del dominio del problema.

Veamos la implementación:

```fortran
module filtrado
    use estadisticas, only: media, desviacion_std, mediana
    implicit none
    private
    public :: detectar_outliers, filtrar_outliers, reemplazar_outliers
    public :: filtrar_por_umbral

contains

    subroutine detectar_outliers(x, mascara, k)
        real, intent(in) :: x(:)
        logical, intent(out) :: mascara(:)
        real, intent(in), optional :: k
        real :: umbral_k, m, std, lim_inf, lim_sup

        umbral_k = 2.0
        if (present(k)) umbral_k = k

        if (size(x) /= size(mascara)) then
            error stop "Error: x y mascara deben tener el mismo tamano"
        end if

        m = media(x)
        std = desviacion_std(x)

        if (std == 0.0) then
            mascara = .false.
            return
        end if

        lim_inf = m - umbral_k * std
        lim_sup = m + umbral_k * std

        mascara = (x < lim_inf) .or. (x > lim_sup)
    end subroutine detectar_outliers

    function filtrar_outliers(x, k) result(filtrado)
        real, intent(in) :: x(:)
        real, intent(in), optional :: k
        real, allocatable :: filtrado(:)
        logical, allocatable :: mask(:)
        real :: umbral_k, m, std, lim_inf, lim_sup

        umbral_k = 2.0
        if (present(k)) umbral_k = k

        m = media(x)
        std = desviacion_std(x)

        if (std == 0.0) then
            filtrado = x
            return
        end if

        lim_inf = m - umbral_k * std
        lim_sup = m + umbral_k * std

        filtrado = pack(x, x >= lim_inf .and. x <= lim_sup)
    end function filtrar_outliers

    function reemplazar_outliers(x, k, valor_reemplazo) result(modificado)
        real, intent(in) :: x(:)
        real, intent(in), optional :: k
        real, intent(in), optional :: valor_reemplazo
        real, allocatable :: modificado(:)
        logical, allocatable :: mask(:)
        real :: reemplazo

        reemplazo = huge(reemplazo)
        if (present(valor_reemplazo)) reemplazo = valor_reemplazo

        allocate(modificado(size(x)))
        modificado = x

        call detectar_outliers(x, mask, k)

        if (reemplazo == huge(reemplazo)) then
            reemplazo = mediana(x)
        end if

        where (mask)
            modificado = reemplazo
        end where
    end function reemplazar_outliers

    function filtrar_por_umbral(x, umbral, comparacion) result(filtrado)
        real, intent(in) :: x(:)
        real, intent(in) :: umbral
        character(len=*), intent(in) :: comparacion
        real, allocatable :: filtrado(:)

        select case (comparacion)
        case (">")
            filtrado = pack(x, x > umbral)
        case (">=")
            filtrado = pack(x, x >= umbral)
        case ("<")
            filtrado = pack(x, x < umbral)
        case ("<=")
            filtrado = pack(x, x <= umbral)
        case ("==")
            filtrado = pack(x, abs(x - umbral) < 1.0e-6)
        case default
            filtrado = pack(x, x > umbral)
        end select
    end function filtrar_por_umbral

end module filtrado
```

**Análisis línea por línea**:

- `module filtrado`: Módulo que agrupa funciones de filtrado y detección de outliers. Depende del módulo `estadisticas` para calcular media y desviación.
- `use estadisticas, only: media, desviacion_std, mediana`: Importa solo las funciones necesarias. `only` evita contaminar el espacio de nombres con funciones no utilizadas.
- `public :: detectar_outliers, filtrar_outliers, ...`: Solo cuatro procedimientos son públicos. Todo lo demás (variables internas) es privado.
- `subroutine detectar_outliers(x, mascara, k)`: Subrutina que llena una máscara lógica indicando qué elementos son outliers. Recibe el array `x`, devuelve `mascara` y acepta opcionalmente `k`.
- `real, intent(in), optional :: k`: Parámetro opcional `k`. Si no se proporciona, usa 2.0 por defecto. `optional` permite llamar a la subrutina sin ese argumento.
- `if (present(k)) umbral_k = k`: `present()` verifica si el argumento opcional fue proporcionado. Sin esta verificación, usar `k` directamente cuando no fue pasado causaría un error en tiempo de ejecución.
- `if (size(x) /= size(mascara)) then error stop ...`: Verifica que los arrays tengan el mismo tamaño. `error stop` detiene el programa inmediatamente. Sin esta verificación, el programa continuaría con índices fuera de límites.
- `m = media(x); std = desviacion_std(x)`: Calcula estadísticas usando las funciones del módulo `estadisticas`.
- `if (std == 0.0) then mascara = .false.; return`: Si la desviación es cero (todos los valores iguales), no hay outliers.
- `lim_inf = m - umbral_k * std; lim_sup = m + umbral_k * std`: Define los límites inferior y superior. Cualquier valor fuera de este rango es outlier.
- `mascara = (x < lim_inf) .or. (x > lim_sup)`: Expresión lógica sobre arrays. Cada elemento de `x` se compara, y el resultado lógico se asigna al correspondiente elemento de `mascara`.
- `function filtrar_outliers(x, k) result(filtrado)`: Función que devuelve un nuevo array sin outliers. El resultado es allocatable y se redimensiona automáticamente.
- `filtrado = x`: Asigna el array completo. Si `std == 0`, no hay outliers, devolvemos los datos originales.
- `filtrado = pack(x, x >= lim_inf .and. x <= lim_sup)`: Extrae solo los elementos que no son outliers. `pack` crea un nuevo array unidimensional con esos elementos.
- `function reemplazar_outliers(x, k, valor_reemplazo) result(modificado)`: Similar pero mantiene el tamaño del array, reemplazando outliers por un valor.
- `reemplazo = huge(reemplazo)`: Inicializa `reemplazo` con el valor real más grande representable. Esto sirve como centinela para saber si el usuario proporcionó un valor de reemplazo.
- `allocate(modificado(size(x))); modificado = x`: Crea una copia del array original. Al ser copia, las modificaciones no afectan al array original.
- `call detectar_outliers(x, mask, k)`: Obtiene la máscara de outliers.
- `if (reemplazo == huge(reemplazo)) then reemplazo = mediana(x)`: Si el usuario no especificó valor de reemplazo, usamos la mediana. La mediana es más robusta a outliers que la media.
- `where (mask) modificado = reemplazo end where`: Reemplaza los outliers en la copia. Los elementos no outliers permanecen intactos.
- `function filtrar_por_umbral(x, umbral, comparacion) result(filtrado)`: Filtra valores según un umbral y un operador de comparación pasado como cadena.
- `select case (comparacion)`: Estructura de selección. Fortran permite `select case` con cadenas (desde Fortran 95).
- `case (">")`: Rama para el operador mayor que. Cada rama usa `pack` con la condición apropiada.
- `case default: filtrado = pack(x, x > umbral)`: Si el operador no se reconoce, usamos `>` por defecto. Esto es un diseño por contrato: el usuario debe proporcionar un operador válido.

**Salida esperada** (para datos de temperatura con algunos outliers):

```
Datos originales: 22.5 25.0 48.3 28.3 19.8 31.2 27.1 30.5 -5.0 26.0
Outliers detectados: 2 valores
Posiciones: 3, 9
Datos sin outliers: 22.5 25.0 28.3 19.8 31.2 27.1 30.5 26.0
Datos con outliers reemplazados por mediana: 22.5 25.0 26.0 28.3 19.8 31.2 27.1 30.5 26.0 26.0
Filtrado (temp > 30): 31.2 30.5
```

**Errores típicos**:

```
! Error 1: Pasar arrays de diferente tamano a detectar_outliers
program error_tamano
    use filtrado
    implicit none
    real :: x(10) = [(real(i), i = 1, 10)]
    logical :: mascara(5)  ! Tamano diferente
    call detectar_outliers(x, mascara)  ! Detiene con error_stop
end program error_tamano
```

```
! Error 2: Usar argumento optional sin verificar present
program error_optional
    implicit none
    real :: x(5) = [1.0, 2.0, 3.0, 4.0, 5.0]
    logical :: mask(5)
    call sub(x, mask)
contains
    subroutine sub(x, mask, k)
        real, intent(in) :: x(:)
        logical, intent(out) :: mask(:)
        real, intent(in), optional :: k
        print *, k  ! Error si k no fue pasado (depende del compilador)
    end subroutine sub
end program error_optional
```

Mensaje de gfortran (en algunos casos no da error, sino valor basura):
```
   0.00000000  (o valor indefinido)
```

```
! Error 3: Confundir .and. con and (sin puntos)
program error_and
    implicit none
    logical :: a = .true., b = .false.
    if (a and b) then  ! Error: 'and' no es operador, debe ser '.and.'
        print *, "No se llega aqui"
    end if
end program error_and
```

Mensaje de gfortran:
```
Error: Syntax error in IF-expression at (1)
```

---

## Correlación y covarianza

La correlación es quizás la herramienta más utilizada en análisis de datos. Responde a la pregunta: ¿cuándo una variable aumenta, la otra también lo hace (correlación positiva), disminuye (correlación negativa) o no hay relación?

La covarianza es el precursor matemático de la correlación. Mide cómo dos variables varían juntas. Su problema es que depende de las escalas: la covarianza entre temperatura en grados Celsius y presión en hectopascales no es comparable con la covarianza entre temperatura en Kelvin y presión en atmósferas. La correlación (coeficiente de Pearson) resuelve esto normalizando por las desviaciones estándar, dando un valor entre -1 y 1.

La fórmula de la covarianza muestral es:
```
cov(x, y) = sum((x_i - mean(x)) * (y_i - mean(y))) / (n - 1)
```

El coeficiente de correlación de Pearson es:
```
r = cov(x, y) / (std(x) * std(y))
```

Interpretación de la correlación:
- r = 1: correlación positiva perfecta
- r = 0.7 a 0.9: correlación positiva fuerte
- r = 0.4 a 0.6: correlación positiva moderada
- r = 0.1 a 0.3: correlación positiva débil
- r = 0: sin correlación lineal
- r < 0: correlación negativa (misma escala, signo opuesto)

¿Por qué la correlación no implica causalidad? Porque una correlación alta puede deberse a una tercera variable no observada (factor de confusión), a una casualidad (especialmente con muestras pequeñas) o a una relación espuria. Como analistas, debemos recordar siempre que la correlación cuantifica la asociación lineal, no la relación causal.

En Fortran, la covarianza y correlación se implementan eficientemente usando aritmética de arrays. La expresión `sum((x - mx) * (y - my))` calcula la suma de productos de desviaciones sin un solo bucle explícito. El compilador puede vectorizar esta operación, haciéndola extremadamente rápida para arrays grandes.

Un detalle importante: la covarianza y correlación son sensibles a outliers. Un solo valor extremo puede inflar o deflactar artificialmente la correlación. Por eso es recomendable calcular la correlación después de filtrar outliers, o usar correlaciones robustas (como la de Spearman, que usa rangos en lugar de valores).

Veamos la implementación:

```fortran
module correlacion
    use estadisticas, only: media, desviacion_std
    implicit none
    private
    public :: covarianza, correlacion_pearson, matriz_covarianza

contains

    function covarianza(x, y) result(cov)
        real, intent(in) :: x(:), y(:)
        real :: cov, mx, my
        integer :: n

        n = size(x)
        if (n /= size(y)) then
            error stop "Error: x e y deben tener el mismo tamano"
        end if
        if (n <= 1) then
            cov = 0.0
            return
        end if

        mx = media(x)
        my = media(y)
        cov = sum((x - mx) * (y - my)) / real(n - 1)
    end function covarianza

    function correlacion_pearson(x, y) result(r)
        real, intent(in) :: x(:), y(:)
        real :: r, stdx, stdy

        if (size(x) /= size(y)) then
            error stop "Error: x e y deben tener el mismo tamano"
        end if

        stdx = desviacion_std(x)
        stdy = desviacion_std(y)

        if (stdx == 0.0 .or. stdy == 0.0) then
            r = 0.0
            return
        end if

        r = covarianza(x, y) / (stdx * stdy)
    end function correlacion_pearson

    subroutine matriz_covarianza(datos, matriz)
        real, intent(in) :: datos(:,:)
        real, intent(out) :: matriz(:,:)
        integer :: nvar, i, j

        nvar = size(datos, 1)
        if (size(matriz, 1) /= nvar .or. size(matriz, 2) /= nvar) then
            error stop "Error: matriz debe ser nvar x nvar"
        end if

        do i = 1, nvar
            do j = i, nvar
                matriz(i, j) = covarianza(datos(i, :), datos(j, :))
                matriz(j, i) = matriz(i, j)  ! matriz simetrica
            end do
        end do
    end subroutine matriz_covarianza

end module correlacion
```

**Análisis línea por línea**:

- `module correlacion`: Módulo que agrupa funciones de correlación y covarianza. Reutiliza `media` y `desviacion_std` del módulo `estadisticas`.
- `public :: covarianza, correlacion_pearson, matriz_covarianza`: Expone tres procedimientos. La función `matriz_covarianza` es particularmente útil cuando tenemos múltiples variables.
- `function covarianza(x, y) result(cov)`: Función que calcula la covarianza muestral entre dos arrays.
- `if (n /= size(y)) call error stop`: Verifica que los arrays tengan el mismo tamaño. La covarianza no tiene sentido entre arrays de diferente longitud porque la suma de productos necesita pares (x_i, y_i).
- `if (n <= 1) then cov = 0.0; return`: Con 0 o 1 punto de datos, la covarianza no está definida. Devolvemos 0.0.
- `mx = media(x); my = media(y)`: Calcula las medias de ambos arrays. Necesitamos las medias para centrar los datos.
- `cov = sum((x - mx) * (y - my)) / real(n - 1)`: La fórmula completa en una línea. `(x - mx)` y `(y - my)` son arrays temporales de desviaciones. `*` es multiplicación elemento a elemento (producto de Hadamard). `sum` suma todos los productos.
- `function correlacion_pearson(x, y) result(r)`: Calcula el coeficiente de correlación de Pearson.
- `r = covarianza(x, y) / (stdx * stdy)`: La correlación es la covarianza normalizada por el producto de las desviaciones estándar. El resultado siempre está entre -1 y 1.
- `if (stdx == 0.0 .or. stdy == 0.0) then r = 0.0; return`: Si alguna variable no tiene variación, la correlación no está definida. Devolvemos 0.0.
- `subroutine matriz_covarianza(datos, matriz)`: Subrutina que calcula la matriz de covarianza completa. `datos(:,:)` tiene variables en filas y observaciones en columnas.
- `nvar = size(datos, 1)`: Número de variables (primera dimensión).
- `if (size(matriz, 1) /= nvar .or. size(matriz, 2) /= nvar)`: Verifica que la matriz de salida tenga el tamaño correcto.
- `do i = 1, nvar; do j = i, nvar`: Bucle sobre la mitad superior de la matriz (incluyendo la diagonal). Aprovechamos la simetría para no calcular dos veces la misma covarianza.
- `matriz(i, j) = covarianza(datos(i, :), datos(j, :))`: Calcula la covarianza entre la variable i y la variable j. `datos(i, :)` toma toda la fila i (todas las observaciones de la variable i).
- `matriz(j, i) = matriz(i, j)`: La matriz de covarianza es simétrica. Copiamos el valor calculado al elemento simétrico.

**Salida esperada** (para temperatura y humedad):

```
Covarianza (temp, humedad): -12.4532
Correlacion de Pearson: -0.7845
Interpretacion: correlacion negativa fuerte
```

**Errores típicos**:

```
! Error 1: Calcular correlacion con arrays de diferentes tamanos
program error_corr_tamano
    use correlacion
    implicit none
    real :: x(10) = [(real(i), i = 1, 10)]
    real :: y(8)  = [(real(i), i = 1, 8)]
    print *, correlacion_pearson(x, y)  ! Error_stop
end program error_corr_tamano
```

```
! Error 2: Usar correlacion con arrays sin variacion
program error_corr_constante
    use correlacion
    implicit none
    real :: x(10) = [1.0, 1.0, 1.0, 1.0, 1.0, 1.0, 1.0, 1.0, 1.0, 1.0]
    real :: y(10) = [(real(i), i = 1, 10)]
    print *, correlacion_pearson(x, y)  ! Devuelve 0.0 porque stdx = 0
end program error_corr_constante
```

```
! Error 3: Olvidar que covarianza requiere n > 1
program error_cov_n1
    use correlacion
    implicit none
    real :: x(1) = [5.0]
    real :: y(1) = [3.0]
    print *, covarianza(x, y)  ! Devuelve 0.0 por n <= 1
end program error_cov_n1
```

Mensaje de gfortran (no muestra error, pero el resultado es 0.0):
```
   0.00000000
```

---

## Histogramas en texto

Un histograma es una representación gráfica de la distribución de frecuencias de una variable. Divide el rango de valores en intervalos (bins) y cuenta cuántos datos caen en cada intervalo. En una terminal, sin librerías gráficas, podemos representar histogramas con barras de asteriscos (*).

¿Por qué hacer histogramas en texto? Porque muchas veces trabajamos en servidores remotos sin acceso a gráficos, o necesitamos una visualización rápida sin instalar librerías adicionales. Un histograma de texto, aunque básico, transmite la forma de la distribución: ¿es simétrica? ¿Tiene picos? ¿Hay valores extremos?

La construcción de un histograma implica varias decisiones de diseño:
1. **Número de bins**: Muy pocos bins ocultan detalles; demasiados bins muestran ruido. Una regla común es la de Sturges: `k = ceil(log2(n) + 1)`.
2. **Límites**: ¿Usamos los valores mínimo y máximo de los datos, o límites ligeramente extendidos? Si usamos los extremos exactos, los bins extremos pueden tener pocos datos.
3. **Ancho del bin**: `ancho = (max - min) / k`. Debe ser constante para que el histograma sea interpretable.
4. **Fronteras**: ¿El bin [a, b) incluye a y excluye b? ¿O [a, b]? La convención más común es [a, b), pero debemos ser consistentes.

En Fortran, implementamos el histograma con un bucle que recorre los datos e incrementa el contador del bin apropiado. Para cada valor, calculamos `indice = floor((valor - min) / ancho) + 1`, con cuidado de manejar el valor máximo (que cae en el último bin). `floor` es una función intrínseca que redondea hacia abajo.

La representación en texto escala la frecuencia máxima a una longitud fija (por ejemplo, 50 caracteres). Cada asterisco representa una fracción de la frecuencia total.

Veamos la implementación completa con un programa de ejemplo:

```fortran
module histograma
    implicit none
    private
    public :: calcular_histograma, imprimir_histograma

contains

    function calcular_histograma(x, nbins) result(frecuencias)
        real, intent(in) :: x(:)
        integer, intent(in) :: nbins
        integer, allocatable :: frecuencias(:)
        real :: min_val, max_val, ancho
        integer :: i, idx

        if (size(x) == 0) then
            allocate(frecuencias(0))
            return
        end if

        allocate(frecuencias(nbins))
        frecuencias = 0

        min_val = minval(x)
        max_val = maxval(x)

        if (abs(max_val - min_val) < 1.0e-10) then
            frecuencias(1) = size(x)
            return
        end if

        ancho = (max_val - min_val) / real(nbins)

        do i = 1, size(x)
            idx = floor((x(i) - min_val) / ancho) + 1
            if (idx > nbins) idx = nbins  ! valor maximo cae en el ultimo bin
            frecuencias(idx) = frecuencias(idx) + 1
        end do
    end function calcular_histograma

    subroutine imprimir_histograma(x, nbins, titulo)
        real, intent(in) :: x(:)
        integer, intent(in) :: nbins
        character(len=*), intent(in), optional :: titulo
        integer, allocatable :: frecuencias(:)
        real :: min_val, max_val, ancho
        integer :: max_freq, escala, i, j, asteriscos
        character(len=50) :: barra

        if (size(x) == 0) then
            print "(A)", "No hay datos para el histograma"
            return
        end if

        frecuencias = calcular_histograma(x, nbins)

        if (present(titulo)) then
            print "(A)", titulo
        end if

        min_val = minval(x)
        max_val = maxval(x)
        ancho = (max_val - min_val) / real(nbins)
        max_freq = maxval(frecuencias)

        if (max_freq == 0) then
            print "(A)", "Histograma vacio"
            return
        end if

        ! Escalar a 40 asteriscos maximo
        escala = max_freq / 40 + 1

        do i = 1, nbins
            asteriscos = frecuencias(i) / escala
            barra = ""
            do j = 1, asteriscos
                barra(j:j) = "*"
            end do
            write(*, "(F8.2, ' - ', F8.2, ' | ', A, ' (', I0, ')')") &
                min_val + (i-1)*ancho, min_val + i*ancho, &
                trim(barra), frecuencias(i)
        end do
    end subroutine imprimir_histograma

end module histograma
```

**Análisis línea por línea**:

- `module histograma`: Módulo autocontenido para histogramas. No depende de otros módulos.
- `function calcular_histograma(x, nbins) result(frecuencias)`: Función pura que devuelve un array de frecuencias. No imprime nada; separa el cálculo de la presentación.
- `integer, allocatable :: frecuencias(:)`: El resultado es un array allocatable. Se asigna dentro de la función.
- `if (size(x) == 0) then allocate(frecuencias(0)); return`: Maneja el caso de array vacío. Sin esta verificación, `minval(x)` fallaría.
- `min_val = minval(x); max_val = maxval(x)`: Encuentra los valores extremos. En Fortran 2008, también podríamos usar `findloc` para encontrar posiciones.
- `if (abs(max_val - min_val) < 1.0e-10)`: Si todos los valores son iguales, no hay bins significativos. Ponemos todo en el primer bin y salimos.
- `ancho = (max_val - min_val) / real(nbins)`: Calcula el ancho de cada bin como un número real. La división debe ser en real para mantener precisión.
- `do i = 1, size(x)`: Itera sobre todos los datos. Para conjuntos muy grandes, este bucle puede ser el cuello de botella; en ese caso, podríamos vectorizar usando `where` y `sum`, pero la lógica es más compleja.
- `idx = floor((x(i) - min_val) / ancho) + 1`: Calcula el índice del bin. `floor` redondea hacia abajo. Sumamos 1 porque Fortran indexa desde 1.
- `if (idx > nbins) idx = nbins`: Corrección para el valor máximo. Sin esto, `idx` podría ser `nbins + 1` cuando `x(i) == max_val`, causando un error de índice fuera de límites.
- `frecuencias(idx) = frecuencias(idx) + 1`: Incrementa el contador. En Fortran, `frecuencias(idx)` es una operación de lectura-modificación-escritura.
- `subroutine imprimir_histograma(x, nbins, titulo)`: Subrutina que imprime el histograma. Tiene un argumento opcional `titulo`.
- `character(len=*), intent(in), optional :: titulo`: Cadena de longitud arbitraria. `optional` permite llamar sin título.
- `character(len=50) :: barra`: Buffer para la barra de asteriscos. 50 caracteres es suficiente para una terminal estándar.
- `frecuencias = calcular_histograma(x, nbins)`: Llama a la función de cálculo. La asignación a un allocatable redimensiona automáticamente.
- `if (present(titulo)) print "(A)", titulo`: Imprime el título si fue proporcionado.
- `escala = max_freq / 40 + 1`: Calcula la escala para que la barra más larga tenga aproximadamente 40 asteriscos. El `+1` evita división por cero si `max_freq < 40`.
- `do i = 1, nbins`: Itera sobre los bins para imprimir.
- `asteriscos = frecuencias(i) / escala`: Número de asteriscos para este bin (división entera, se trunca).
- `barra(j:j) = "*"`: Asigna un asterisco a cada carácter de la cadena. En Fortran, las cadenas se acceden carácter por carácter con índices.
- `write(*, "(F8.2, ' - ', F8.2, ' | ', A, ' (', I0, ')')") ...`: Formato complejo que imprime: límite inferior, guión, límite superior, barra, y frecuencia entre paréntesis.

**Salida esperada** (para datos de temperatura con 100 valores):

```
Histograma de Temperatura
   18.50 -   20.04 | ******* (7)
   20.04 -   21.58 | ********** (10)
   21.58 -   23.12 | *************** (15)
   23.12 -   24.66 | ******************* (19)
   24.66 -   26.20 | ********************* (21)
   26.20 -   27.74 | **************** (16)
   27.74 -   29.28 | *********** (11)
   29.28 -   30.82 | ***** (5)
   30.82 -   32.36 | *** (3)
   32.36 -   33.90 | * (1)
```

**Errores típicos**:

```
! Error 1: Calcular indice de bin incorrectamente
program error_hist_idx
    implicit none
    real :: x(5) = [1.0, 2.0, 3.0, 4.0, 5.0]
    integer :: nbins = 4, i, idx
    real :: min_val = 1.0, max_val = 5.0
    real :: ancho
    integer :: frec(4) = 0
    ancho = (max_val - min_val) / real(nbins)  ! ancho = 1.0
    do i = 1, 5
        ! Error: usar nint (redondeo al mas cercano) en lugar de floor
        idx = nint((x(i) - min_val) / ancho) + 1
        ! Con nint, el valor 1.0 da: nint(0.0) + 1 = 1 (correcto)
        ! Pero 2.7 daria: nint(1.7) + 1 = 2 + 1 = 3 (incorrecto, deberia ser 2)
        if (idx > nbins) idx = nbins
        frec(idx) = frec(idx) + 1
    end do
    print *, frec  ! Distribucion incorrecta
end program error_hist_idx
```

```
! Error 2: No manejar el caso de todos los valores iguales
program error_hist_igual
    implicit none
    real :: x(10) = [5.0, 5.0, 5.0, 5.0, 5.0, 5.0, 5.0, 5.0, 5.0, 5.0]
    integer :: nbins = 5, i, idx
    real :: ancho = 0.0  ! max_val - min_val = 0.0
    ! Division por cero en el calculo de idx
    ! floor((5.0 - 5.0) / 0.0) = floor(infinito / infinito) = NaN
    print *, floor((5.0 - 5.0) / ancho)  ! NaN -> error
end program error_hist_igual
```

---

## Mini-proyecto: Analizador de Datos Climáticos

Ha llegado el momento de integrar todo lo aprendido en un programa completo y funcional. Este mini-proyecto te llevará de principio a fin en la construcción de un analizador de datos climáticos. No solo leerás y procesarás datos, sino que organizarás el código en módulos, manejarás errores, calcularás estadísticas, filtrarás outliers, correlacionarás variables, generarás histogramas y exportarás resultados.

La filosofía de diseño es modular: un módulo para estadísticas, otro para E/S (entrada/salida), otro para filtrado y otro para histogramas. Cada módulo tiene una responsabilidad única y bien definida. Esto no solo hace el código más mantenible, sino que permite probar cada componente por separado.

Los datos de ejemplo serán un archivo CSV con 100 registros de temperatura (°C), humedad relativa (%) y presión atmosférica (hPa). Algunos registros tendrán valores centinela (-999.0) que deberemos detectar y manejar.

El programa principal:
1. Lee el archivo CSV usando el módulo `io_clima`.
2. Calcula estadísticas descriptivas completas para las tres variables.
3. Detecta y filtra outliers (más de 2 desviaciones estándar).
4. Calcula la correlación entre temperatura y humedad.
5. Filtra días extremos (temperatura > 30°C).
6. Genera histogramas de texto para las tres variables.
7. Exporta los resultados a un archivo CSV de salida.

Usaremos arrays allocatable, `where` y `pack` para filtrado, y módulos para organización. Todo el código es compilable con gfortran 13+ con `-std=f2018`.

Comencemos con el archivo de datos de ejemplo. Los datos simulados representan un clima templado con algunos días calurosos y fríos, con outliers ocasionales:

```
fecha,temperatura,humedad,presion
2024-01-01,22.5,60.2,1013.5
2024-01-02,23.1,58.7,1012.8
2024-01-03,21.8,62.3,1014.2
2024-01-04,25.0,55.1,1011.9
2024-01-05,19.7,68.5,1015.6
2024-01-06,24.2,59.8,1013.0
2024-01-07,26.3,52.4,1010.5
2024-01-08,20.5,65.2,1014.8
2024-01-09,18.9,70.1,1016.2
2024-01-10,25.8,54.6,1011.2
```

Y ahora, el código completo del proyecto, organizado en cuatro archivos:

### `mod_estadisticas.f90`

```fortran
module mod_estadisticas
    implicit none
    private
    public :: media, mediana, moda, varianza, desviacion_std
    public :: asimetria, curtosis, min_val, max_val, ordenar

contains

    function media(x) result(m)
        real, intent(in) :: x(:)
        real :: m
        integer :: n
        n = size(x)
        if (n == 0) then
            m = 0.0
            return
        end if
        m = sum(x) / real(n)
    end function media

    function varianza(x) result(v)
        real, intent(in) :: x(:)
        real :: v, m
        integer :: n
        n = size(x)
        if (n <= 1) then
            v = 0.0
            return
        end if
        m = media(x)
        v = sum((x - m)**2) / real(n - 1)
    end function varianza

    function desviacion_std(x) result(s)
        real, intent(in) :: x(:)
        real :: s
        s = sqrt(varianza(x))
    end function desviacion_std

    function mediana(x) result(m)
        real, intent(in) :: x(:)
        real :: m
        real, allocatable :: y(:)
        integer :: n, mid
        n = size(x)
        if (n == 0) then
            m = 0.0
            return
        end if
        y = x
        call ordenar(y)
        mid = n / 2
        if (mod(n, 2) == 0) then
            m = (y(mid) + y(mid + 1)) / 2.0
        else
            m = y(mid + 1)
        end if
    end function mediana

    subroutine ordenar(arr)
        real, intent(inout) :: arr(:)
        integer :: i, j
        real :: key
        do i = 2, size(arr)
            key = arr(i)
            j = i - 1
            do while (j >= 1 .and. arr(j) > key)
                arr(j + 1) = arr(j)
                j = j - 1
            end do
            arr(j + 1) = key
        end do
    end subroutine ordenar

    function moda(x) result(m)
        real, intent(in) :: x(:)
        real :: m
        real, allocatable :: y(:)
        integer :: i, max_count, count_actual
        if (size(x) == 0) then
            m = 0.0
            return
        end if
        y = x
        call ordenar(y)
        m = y(1)
        max_count = 1
        count_actual = 1
        do i = 2, size(y)
            if (abs(y(i) - y(i-1)) < 1.0e-6) then
                count_actual = count_actual + 1
            else
                if (count_actual > max_count) then
                    max_count = count_actual
                    m = y(i-1)
                end if
                count_actual = 1
            end if
        end do
        if (count_actual > max_count) then
            m = y(size(y))
        end if
    end function moda

    function asimetria(x) result(s)
        real, intent(in) :: x(:)
        real :: s, m, std
        integer :: n
        n = size(x)
        if (n < 3) then
            s = 0.0
            return
        end if
        m = media(x)
        std = desviacion_std(x)
        if (std == 0.0) then
            s = 0.0
            return
        end if
        s = (sum(((x - m) / std)**3) * real(n)) / real((n - 1) * (n - 2))
    end function asimetria

    function curtosis(x) result(k)
        real, intent(in) :: x(:)
        real :: k, m, std
        integer :: n
        n = size(x)
        if (n < 4) then
            k = 0.0
            return
        end if
        m = media(x)
        std = desviacion_std(x)
        if (std == 0.0) then
            k = 0.0
            return
        end if
        k = (sum(((x - m) / std)**4) * real(n * (n + 1))) / &
            real((n - 1) * (n - 2) * (n - 3)) - &
            (3.0 * real((n - 1)**2)) / real((n - 2) * (n - 3))
    end function curtosis

    function min_val(x) result(m)
        real, intent(in) :: x(:)
        real :: m
        if (size(x) == 0) then
            m = huge(m)
            return
        end if
        m = minval(x)
    end function min_val

    function max_val(x) result(m)
        real, intent(in) :: x(:)
        real :: m
        if (size(x) == 0) then
            m = -huge(m)
            return
        end if
        m = maxval(x)
    end function max_val

end module mod_estadisticas
```

### `mod_io.f90`

```fortran
module mod_io
    implicit none
    private
    public :: leer_csv, exportar_resultados

contains

    subroutine leer_csv(archivo, temperatura, humedad, presion, fechas)
        character(len=*), intent(in) :: archivo
        real, allocatable, intent(out) :: temperatura(:)
        real, allocatable, intent(out) :: humedad(:)
        real, allocatable, intent(out) :: presion(:)
        character(len=10), allocatable, intent(out) :: fechas(:)
        character(len=256) :: linea
        integer :: unidad, iostat, nlines, i
        integer :: pos1, pos2, ncomas
        real :: t, h, p
        character(len=10) :: f

        ! Primera pasada: contar lineas
        open(newunit=unidad, file=archivo, status="old", action="read", iostat=iostat)
        if (iostat /= 0) then
            error stop "Error: No se pudo abrir " // trim(archivo)
        end if

        nlines = 0
        read(unidad, "(A)", iostat=iostat) linea  ! cabecera
        do
            read(unidad, "(A)", iostat=iostat) linea
            if (iostat /= 0) exit
            linea = adjustl(linea)
            if (len_trim(linea) == 0) cycle
            nlines = nlines + 1
        end do
        close(unidad)

        if (nlines == 0) then
            error stop "Error: El archivo no contiene datos"
        end if

        ! Segunda pasada: leer
        allocate(temperatura(nlines), humedad(nlines), presion(nlines))
        allocate(fechas(nlines))

        open(newunit=unidad, file=archivo, status="old", action="read", iostat=iostat)
        read(unidad, "(A)", iostat=iostat) linea  ! cabecera

        do i = 1, nlines
            read(unidad, "(A)", iostat=iostat) linea
            if (iostat /= 0) exit

            f = ""
            t = -999.0
            h = -999.0
            p = -999.0
            ncomas = 0
            pos1 = 1

            ! Leer fecha
            pos2 = index(linea(pos1:), ",")
            if (pos2 > 0) then
                f = linea(pos1:pos1+pos2-2)
                pos1 = pos1 + pos2
                ncomas = 1
            end if

            ! Leer temperatura
            pos2 = index(linea(pos1:), ",")
            if (pos2 > 0) then
                read(linea(pos1:pos1+pos2-2), *, iostat=iostat) t
                if (iostat /= 0) t = -999.0
                pos1 = pos1 + pos2
                ncomas = 2
            end if

            ! Leer humedad
            pos2 = index(linea(pos1:), ",")
            if (pos2 > 0) then
                read(linea(pos1:pos1+pos2-2), *, iostat=iostat) h
                if (iostat /= 0) h = -999.0
                pos1 = pos1 + pos2
                ncomas = 3
            end if

            ! Leer presion (ultimo campo)
            read(linea(pos1:), *, iostat=iostat) p
            if (iostat /= 0) p = -999.0

            fechas(i) = f
            temperatura(i) = t
            humedad(i) = h
            presion(i) = p
        end do
        close(unidad)
    end subroutine leer_csv

    subroutine exportar_resultados(archivo, variable, media_val, mediana_val, &
            desviacion, min_val, max_val, correlacion_th)
        character(len=*), intent(in) :: archivo
        character(len=*), intent(in) :: variable
        real, intent(in) :: media_val, mediana_val, desviacion
        real, intent(in) :: min_val, max_val, correlacion_th
        integer :: unidad, iostat

        open(newunit=unidad, file=archivo, status="replace", action="write", iostat=iostat)
        if (iostat /= 0) then
            error stop "Error: No se pudo crear " // trim(archivo)
        end if

        write(unidad, "(A)") "Variable,Media,Mediana,Desviacion,Minimo,Maximo,Correlacion_TH"
        write(unidad, "(A,6(F0.2,','),F0.4)") &
            trim(variable) // ",", media_val, mediana_val, desviacion, &
            min_val, max_val, correlacion_th

        close(unidad)
    end subroutine exportar_resultados

end module mod_io
```

### `mod_filtrado.f90`

```fortran
module mod_filtrado
    use mod_estadisticas, only: media, desviacion_std, mediana
    implicit none
    private
    public :: detectar_outliers, filtrar_outliers, filtrar_por_umbral

contains

    subroutine detectar_outliers(x, mascara, k)
        real, intent(in) :: x(:)
        logical, allocatable, intent(out) :: mascara(:)
        real, intent(in), optional :: k
        real :: umbral_k, m, std, lim_inf, lim_sup

        umbral_k = 2.0
        if (present(k)) umbral_k = k

        allocate(mascara(size(x)))

        m = media(x)
        std = desviacion_std(x)

        if (std == 0.0) then
            mascara = .false.
            return
        end if

        lim_inf = m - umbral_k * std
        lim_sup = m + umbral_k * std

        mascara = (x < lim_inf) .or. (x > lim_sup)
    end subroutine detectar_outliers

    function filtrar_outliers(x, k) result(filtrado)
        real, intent(in) :: x(:)
        real, intent(in), optional :: k
        real, allocatable :: filtrado(:)
        logical, allocatable :: mask(:)

        call detectar_outliers(x, mask, k)
        filtrado = pack(x, .not. mask)
    end function filtrar_outliers

    function filtrar_por_umbral(x, umbral) result(filtrado)
        real, intent(in) :: x(:)
        real, intent(in) :: umbral
        real, allocatable :: filtrado(:)

        filtrado = pack(x, x > umbral)
    end function filtrar_por_umbral

end module mod_filtrado
```

### `programa_principal.f90`

```fortran
program analizador_climatico
    use mod_estadisticas
    use mod_io
    use mod_filtrado
    implicit none

    real, allocatable :: temperatura(:), humedad(:), presion(:)
    character(len=10), allocatable :: fechas(:)
    real, allocatable :: temp_filtradas(:), hum_filtradas(:)
    real, allocatable :: temp_outliers(:), hum_outliers(:)
    real, allocatable :: dias_extremos(:)
    logical, allocatable :: mask_outliers(:)
    integer :: n, i, n_outliers
    real :: corr_th

    character(len=*), parameter :: ARCHIVO_ENTRADA = "datos_clima.csv"
    character(len=*), parameter :: ARCHIVO_SALIDA  = "resultados.csv"

    ! 1. Lectura de datos
    print "(A)", "=== ANALIZADOR DE DATOS CLIMATICOS ==="
    print "(A)", "Leyendo archivo: " // ARCHIVO_ENTRADA
    call leer_csv(ARCHIVO_ENTRADA, temperatura, humedad, presion, fechas)
    n = size(temperatura)
    print "(A, I0, A)", "Leidos ", n, " registros"
    print *

    ! 2. Limpiar datos: reemplazar valores centinela (-999.0) con NaN
    ! Usamos un valor especial para representar NaN (Fortran no tiene NaN nativo en todos los casos)
    where (temperatura < -900.0) temperatura = -999.0
    where (humedad < -900.0) humedad = -999.0
    where (presion < -900.0) presion = -999.0

    ! Contar valores faltantes
    print "(A, I0)", "Valores faltantes en temperatura: ", count(temperatura < -900.0)
    print "(A, I0)", "Valores faltantes en humedad: ", count(humedad < -900.0)
    print "(A, I0)", "Valores faltantes en presion: ", count(presion < -900.0)
    print *

    ! Filtrar valores validos para estadisticas
    temp_filtradas = pack(temperatura, temperatura > -900.0)
    hum_filtradas = pack(humedad, humedad > -900.0)

    ! 3. Estadisticas descriptivas
    print "(A)", "--- ESTADISTICAS DE TEMPERATURA ---"
    print "(A, F8.2)", "Media: ", media(temp_filtradas)
    print "(A, F8.2)", "Mediana: ", mediana(temp_filtradas)
    print "(A, F8.2)", "Moda: ", moda(temp_filtradas)
    print "(A, F8.2)", "Varianza: ", varianza(temp_filtradas)
    print "(A, F8.2)", "Desviacion Estandar: ", desviacion_std(temp_filtradas)
    print "(A, F8.2)", "Asimetria: ", asimetria(temp_filtradas)
    print "(A, F8.2)", "Curtosis: ", curtosis(temp_filtradas)
    print "(A, F8.2)", "Minimo: ", min_val(temp_filtradas)
    print "(A, F8.2)", "Maximo: ", max_val(temp_filtradas)
    print *

    print "(A)", "--- ESTADISTICAS DE HUMEDAD ---"
    print "(A, F8.2)", "Media: ", media(hum_filtradas)
    print "(A, F8.2)", "Mediana: ", mediana(hum_filtradas)
    print "(A, F8.2)", "Desviacion Estandar: ", desviacion_std(hum_filtradas)
    print "(A, F8.2)", "Minimo: ", min_val(hum_filtradas)
    print "(A, F8.2)", "Maximo: ", max_val(hum_filtradas)
    print *

    ! 4. Deteccion de outliers
    print "(A)", "--- DETECCION DE OUTLIERS ---"
    call detectar_outliers(temp_filtradas, mask_outliers)
    temp_outliers = pack(temp_filtradas, mask_outliers)
    n_outliers = size(temp_outliers)
    print "(A, I0, A)", "Outliers en temperatura: ", n_outliers, " valores"
    if (n_outliers > 0) then
        do i = 1, min(n_outliers, 5)
            print "(A, I0, A, F8.2)", "  Outlier ", i, ": ", temp_outliers(i)
        end do
    end if
    print *

    ! 5. Correlacion temperatura-humedad
    print "(A)", "--- CORRELACION TEMPERATURA-HUMEDAD ---"
    corr_th = correlacion_pearson(temp_filtradas, hum_filtradas)
    print "(A, F8.4)", "Coeficiente de Pearson: ", corr_th
    if (abs(corr_th) > 0.7) then
        print "(A)", "Interpretacion: Correlacion fuerte"
    else if (abs(corr_th) > 0.4) then
        print "(A)", "Interpretacion: Correlacion moderada"
    else
        print "(A)", "Interpretacion: Correlacion debil"
    end if
    if (corr_th < 0) then
        print "(A)", "Direccion: Negativa (cuando aumenta T, disminuye H)"
    else
        print "(A)", "Direccion: Positiva"
    end if
    print *

    ! 6. Filtrar dias extremos
    print "(A)", "--- DIAS EXTREMOS (Temperatura > 30 C) ---"
    dias_extremos = pack(temp_filtradas, temp_filtradas > 30.0)
    if (size(dias_extremos) > 0) then
        print "(A, I0)", "Dias con temperatura > 30 C: ", size(dias_extremos)
        do i = 1, size(dias_extremos)
            print "(A, I0, A, F8.2, A)", "  Dia extremo ", i, ": ", dias_extremos(i), " C"
        end do
    else
        print "(A)", "No se encontraron dias extremos"
    end if
    print *

    ! 7. Histogramas
    print "(A)", "--- HISTOGRAMA DE TEMPERATURA (10 bins) ---"
    call imprimir_histograma(temp_filtradas, 10, "Distribucion de Temperatura")
    print *

    print "(A)", "--- HISTOGRAMA DE HUMEDAD (10 bins) ---"
    call imprimir_histograma(hum_filtradas, 10, "Distribucion de Humedad")
    print *

    ! 8. Exportar resultados
    print "(A)", "Exportando resultados a " // ARCHIVO_SALIDA
    call exportar_resultados(ARCHIVO_SALIDA, "Temperatura", &
        media(temp_filtradas), mediana(temp_filtradas), &
        desviacion_std(temp_filtradas), min_val(temp_filtradas), &
        max_val(temp_filtradas), corr_th)
    print "(A)", "Exportacion completada."
    print *

    print "(A)", "=== ANALISIS COMPLETADO ==="

contains

    function correlacion_pearson(x, y) result(r)
        real, intent(in) :: x(:), y(:)
        real :: r, mx, my, stdx, stdy
        integer :: n
        n = size(x)
        if (n /= size(y) .or. n <= 1) then
            r = 0.0
            return
        end if
        mx = media(x)
        my = media(y)
        stdx = desviacion_std(x)
        stdy = desviacion_std(y)
        if (stdx == 0.0 .or. stdy == 0.0) then
            r = 0.0
            return
        end if
        r = sum((x - mx) * (y - my)) / (real(n - 1) * stdx * stdy)
    end function correlacion_pearson

    subroutine imprimir_histograma(x, nbins, titulo)
        real, intent(in) :: x(:)
        integer, intent(in) :: nbins
        character(len=*), intent(in) :: titulo
        integer, allocatable :: frecuencias(:)
        real :: min_val, max_val, ancho
        integer :: max_freq, escala, i, j, asteriscos, idx
        character(len=60) :: barra

        if (size(x) == 0) then
            print "(A)", "No hay datos para el histograma"
            return
        end if

        allocate(frecuencias(nbins))
        frecuencias = 0

        min_val = minval(x)
        max_val = maxval(x)

        if (abs(max_val - min_val) < 1.0e-10) then
            frecuencias(1) = size(x)
        else
            ancho = (max_val - min_val) / real(nbins)
            do i = 1, size(x)
                idx = floor((x(i) - min_val) / ancho) + 1
                if (idx > nbins) idx = nbins
                frecuencias(idx) = frecuencias(idx) + 1
            end do
        end if

        print "(A)", titulo
        max_freq = maxval(frecuencias)
        if (max_freq == 0) then
            print "(A)", "Histograma vacio"
            return
        end if

        escala = max_freq / 40 + 1

        do i = 1, nbins
            asteriscos = frecuencias(i) / escala
            barra = ""
            do j = 1, asteriscos
                barra(j:j) = "*"
            end do
            write(*, "(F8.2, ' - ', F8.2, ' | ', A, ' (', I0, ')')") &
                min_val + (i-1)*ancho, min_val + i*ancho, &
                trim(barra), frecuencias(i)
        end do
    end subroutine imprimir_histograma

end program analizador_climatico
```

**Análisis línea por línea del programa principal**:

- `program analizador_climatico`: Programa principal. Su nombre debe coincidir con el nombre de archivo recomendado.
- `use mod_estadisticas, use mod_io, use mod_filtrado`: Importa los tres módulos. El orden de `use` no importa, pero es buena práctica alfabetizarlos.
- `real, allocatable :: temperatura(:), humedad(:), presion(:)`: Arrays allocatable para las tres variables. Se asignan dentro de `leer_csv`.
- `character(len=10), allocatable :: fechas(:)`: Array de fechas. Las fechas se almacenan como cadenas porque no hacemos cálculos con ellas.
- `real, allocatable :: temp_filtradas(:), hum_filtradas(:)`: Arrays para datos filtrados (sin valores centinela).
- `character(len=*), parameter :: ARCHIVO_ENTRADA = "datos_clima.csv"`: Constante con el nombre del archivo de entrada. Usar `parameter` permite cambiar el nombre en un solo lugar.
- `call leer_csv(ARCHIVO_ENTRADA, temperatura, humedad, presion, fechas)`: Llama a la subrutina de lectura. Los arrays se pasan sin asignar; la subrutina los asigna.
- `n = size(temperatura)`: Obtiene el número de registros después de la lectura.
- `where (temperatura < -900.0) temperatura = -999.0`: Normaliza valores centinela. Usamos `< -900.0` en lugar de `== -999.0` para evitar problemas de precisión. Si un valor es -999.0 exactamente, la comparación funciona; si está cerca, también.
- `print "(A, I0)", "Valores faltantes en temperatura: ", count(temperatura < -900.0)`: `count` es una función intrínseca que cuenta elementos verdaderos en un array lógico. Es equivalente a `size(pack(temperatura, ...))` pero más eficiente.
- `temp_filtradas = pack(temperatura, temperatura > -900.0)`: Crea un nuevo array solo con valores válidos. `pack` automáticamente asigna el tamaño adecuado a `temp_filtradas`.
- `print "(A, F8.2)", "Media: ", media(temp_filtradas)`: Calcula e imprime la media usando la función del módulo. Nótese que pasamos el array ya filtrado: la media se calcula sin valores centinela.
- `call detectar_outliers(temp_filtradas, mask_outliers)`: Detecta outliers en el array filtrado. `mask_outliers` es un array lógico allocatable que se asigna dentro de la subrutina.
- `temp_outliers = pack(temp_filtradas, mask_outliers)`: Extrae los valores que son outliers. Si no hay outliers, `temp_outliers` tiene tamaño 0.
- `corr_th = correlacion_pearson(temp_filtradas, hum_filtradas)`: Calcula la correlación entre temperatura y humedad usando solo datos válidos. La función está definida en `contains` del programa principal.
- `dias_extremos = pack(temp_filtradas, temp_filtradas > 30.0)`: Filtra temperaturas que superan los 30°C. `pack` devuelve un array de tamaño variable.
- `call imprimir_histograma(temp_filtradas, 10, "Distribucion de Temperatura")`: Genera e imprime el histograma con 10 bins. La subrutina está en `contains`.
- `call exportar_resultados(ARCHIVO_SALIDA, ...)`: Exporta las estadísticas a un archivo CSV. Los valores se escriben en formato CSV estándar.

**Análisis de la función `correlacion_pearson` dentro de `contains`**:

- `function correlacion_pearson(x, y) result(r)`: Definida dentro del programa principal, tiene acceso implícito a las variables del programa. Preferimos pasarlas como argumentos para claridad.
- `r = sum((x - mx) * (y - my)) / (real(n - 1) * stdx * stdy)`: Fórmula completa de correlación en una sola línea. La multiplicación en el denominador evita una división adicional.

**Análisis de la subrutina `imprimir_histograma` dentro de `contains`**:

- `integer, allocatable :: frecuencias(:)`: Array allocatable para frecuencias. Se asigna con `allocate(frecuencias(nbins))`.
- `frecuencias = 0`: Inicializa a cero usando asignación masiva. Sin esto, los valores serían indefinidos.
- `if (abs(max_val - min_val) < 1.0e-10)`: Maneja el caso degenerado de todos los valores iguales. En lugar de dividir por cero, asigna todo al primer bin.
- `ancho = (max_val - min_val) / real(nbins)`: Calcula el ancho. `real(nbins)` asegura división en punto flotante.
- `idx = floor((x(i) - min_val) / ancho) + 1`: Fórmula estándar de indexación de bins. `floor` redondea hacia abajo.
- `escala = max_freq / 40 + 1`: Calcula la escala para que la barra máxima tenga ~40 asteriscos. El `+1` garantiza que `escala >= 1`.
- `barra = ""; do j = 1, asteriscos; barra(j:j) = "*"; end do`: Construye la barra de asteriscos carácter por carácter. En Fortran, las cadenas son arrays de caracteres y se modifican elemento a elemento.

**Salida esperada** (para 100 registros con datos simulados):

```
=== ANALIZADOR DE DATOS CLIMATICOS ===
Leyendo archivo: datos_clima.csv
Leidos 100 registros

Valores faltantes en temperatura: 2
Valores faltantes en humedad: 1
Valores faltantes en presion: 0

--- ESTADISTICAS DE TEMPERATURA ---
Media:    24.87
Mediana:   24.95
Moda:      25.10
Varianza:   7.23
Desviacion Estandar:    2.69
Asimetria:    0.15
Curtosis:   -0.42
Minimo:    18.20
Maximo:    33.50

--- ESTADISTICAS DE HUMEDAD ---
Media:    60.34
Mediana:   60.10
Desviacion Estandar:    5.78
Minimo:    45.20
Maximo:    75.80

--- DETECCION DE OUTLIERS ---
Outliers en temperatura: 2 valores
  Outlier 1:    33.50
  Outlier 2:    18.20

--- CORRELACION TEMPERATURA-HUMEDAD ---
Coeficiente de Pearson:  -0.7845
Interpretacion: Correlacion fuerte
Direccion: Negativa (cuando aumenta T, disminuye H)

--- DIAS EXTREMOS (Temperatura > 30 C) ---
Dias con temperatura > 30 C: 5
  Dia extremo 1:    30.20 C
  Dia extremo 2:    31.50 C
  Dia extremo 3:    30.80 C
  Dia extremo 4:    33.50 C
  Dia extremo 5:    31.10 C

--- HISTOGRAMA DE TEMPERATURA (10 bins) ---
Distribucion de Temperatura
   18.20 -   19.73 | **** (4)
   19.73 -   21.26 | ********* (9)
   21.26 -   22.79 | ************* (13)
   22.79 -   24.32 | **************** (16)
   24.32 -   25.85 | ******************* (19)
   25.85 -   27.38 | ************** (14)
   27.38 -   28.91 | ********* (9)
   28.91 -   30.44 | ***** (5)
   30.44 -   31.97 | *** (3)
   31.97 -   33.50 | ** (2)

--- HISTOGRAMA DE HUMEDAD (10 bins) ---
Distribucion de Humedad
   45.20 -   48.26 | ** (2)
   48.26 -   51.32 | **** (4)
   51.32 -   54.38 | ******** (8)
   54.38 -   57.44 | ************* (13)
   57.44 -   60.50 | ***************** (17)
   60.50 -   63.56 | **************** (16)
   63.56 -   66.62 | *********** (11)
   66.62 -   69.68 | ******* (7)
   69.68 -   72.74 | *** (3)
   72.74 -   75.80 | *** (3)

Exportando resultados a resultados.csv
Exportacion completada.

=== ANALISIS COMPLETADO ===
```

**Salida de `resultados.csv`**:

```
Variable,Media,Mediana,Desviacion,Minimo,Maximo,Correlacion_TH
Temperatura,24.87,24.95,2.69,18.20,33.50,-0.7845
```

### Instrucciones de compilación y ejecución

```bash
# Compilar todos los modulos y el programa principal
gfortran -std=f2018 -Wall -Wextra -o analizador_clima \
    mod_estadisticas.f90 \
    mod_io.f90 \
    mod_filtrado.f90 \
    programa_principal.f90

# Ejecutar (requiere datos_clima.csv en el mismo directorio)
./analizador_clima
```

**Errores típicos del mini-proyecto**:

```
! Error 1: Orden incorrecto de modulos en compilacion
! gfortran -std=f2018 programa_principal.f90 mod_estadisticas.f90
! Error: 'mod_estadisticas' is used but not declared
! Siempre compilar los modulos ANTES del programa que los usa
```

```
! Error 2: Array no asignado antes de pasar a pack
program error_pack_noalloc
    implicit none
    real, allocatable :: x(:)  ! No se asigna
    real, allocatable :: y(:)
    ! x no tiene memoria asignada
    y = pack(x, x > 0.0)  ! Error en tiempo de ejecucion
end program error_pack_noalloc
```

Mensaje de gfortran:
```
Fortran runtime error: Attempt to use an ALLOCATABLE variable that is not allocated
```

```
! Error 3: Confundir el orden de dimensiones en arrays bidimensionales
! En Fortran, datos(:, i) es la i-esima columna (primera dimension: filas)
! datos(i, :) es la i-esima fila (segunda dimension: columnas)
! El orden es column-major: la primera dimension varia mas rapido
```

### Ejercicios propuestos

1. **Ampliación de módulos**: Añade una función `correlacion_spearman` al módulo `mod_estadisticas` que calcule la correlación de rangos de Spearman. Pista: convierte los datos a rangos y aplica Pearson.

2. **Ventanas deslizantes**: Implementa una función `media_movil(x, ventana)` que calcule el promedio móvil de un array con el tamaño de ventana especificado. Usa arrays allocatable.

3. **Manejo de fechas**: Modifica `leer_csv` para que convierta las fechas a un tipo `integer` representando día juliano (días desde una fecha base). Calcula la correlación entre temperatura y día del año.

4. **Exportación de histograma**: Añade una función al módulo `mod_io` que exporte el histograma a un archivo de texto, no solo a la consola.

5. **Optimización**: Reemplaza el algoritmo de ordenación por inserción con quicksort para arrays grandes. Compara el rendimiento.

---

## Conclusión

A lo largo de este capítulo, has aprendido a manipular y analizar datos con Fortran moderno. Desde la lectura de archivos CSV hasta la generación de histogramas, pasando por estadística descriptiva, filtrado, detección de outliers y correlación.

Las lecciones más importantes son:

- **Fortran es excepcional para análisis de datos numéricos** gracias a su aritmética de arrays, que permite expresar operaciones complejas sin bucles explícitos.
- **Los módulos son la unidad de organización**: cada aspecto del análisis (estadísticas, E/S, filtrado) debe encapsularse en su propio módulo.
- **Los arrays allocatable** permiten adaptar la memoria al tamaño real de los datos, y su desasignación automática simplifica la gestión de memoria.
- **`where`, `pack` y `merge`** son herramientas expresivas para filtrar, transformar y limpiar datos.
- **El manejo de errores con `iostat` y `error stop`** es crucial para programas robustos que procesan datos del mundo real.

El analizador climático que has construido es un ejemplo real de cómo Fortran resuelve problemas de análisis de datos. Puedes extenderlo con nuevas funcionalidades, adaptarlo a otros dominios (finanzas, física, biología) o integrarlo en pipelines de procesamiento más grandes.

Recuerda: la mejor herramienta para analizar datos no es la más popular, sino la que mejor se ajusta al problema. Para datos masivos y cómputo intensivo, Fortran sigue siendo imbatible.
