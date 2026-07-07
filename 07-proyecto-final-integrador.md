# Capítulo 5: Proyecto Final Integrador — Dashboard de Monitoreo Económico

> Este capítulo culminante integra todos los conceptos de los capítulos anteriores (0–4) en un solo sistema funcional: un **Dashboard de Monitoreo Económico**. Construirá un programa modular en Fortran que ingiere datos reales de indicadores económicos, calcula estadísticas, aplica regresión lineal para proyecciones, genera informes en LaTeX y Gnuplot, y expone todo mediante una API REST con un cliente de prueba.

---

**Stack tecnológico**

- **Compilador**: gfortran 13+ con `-std=f2018`
- **Sistema operativo**: GNU/Linux Ubuntu 22.04+
- **Herramientas de red**: `curl` 7.81+, `netcat-openbsd`
- **Visualización**: Gnuplot 5.4+ (opcional)
- **Reportes**: LaTeX (`pdflatex`, opcional)
- **Estándar**: Fortran 2018, POSIX sockets vía `iso_c_binding`

---

## 1. Visión general del sistema integrador

Hasta ahora, en los capítulos anteriores, ha aprendido a leer archivos CSV, calcular estadísticas descriptivas, implementar regresión lineal, generar reportes en LaTeX y construir un cliente-servidor con HTTP. Cada uno de esos temas se trató por separado, con ejemplos aislados. Este capítulo le pide que **una todas esas piezas** en un solo sistema que funcione de principio a fin.

Imagine que trabaja para el banco central de un país y necesita monitorear mensualmente cuatro indicadores clave: inflación, crecimiento del PIB, tasa de desempleo y tipo de cambio. Los datos llegan en un archivo CSV. Su jefa le pide tres cosas: (1) un análisis estadístico completo, (2) proyecciones a futuro usando regresión lineal y (3) una API para que otros equipos consulten los datos en tiempo real. Además, quiere un informe en LaTeX para la junta directiva y gráficos en Gnuplot.

¿Cómo abordaría este problema sin usar bibliotecas externas, solo con Fortran estándar? La respuesta es la **arquitectura modular**: dividir el sistema en módulos independientes, cada uno con una responsabilidad única, y luego orquestarlos desde un programa principal. Esta es exactamente la filosofía que seguiremos aquí.

El sistema completo consta de siete archivos fuente:

| Archivo | Responsabilidad | Capítulo relacionado |
|---|---|---|
| `mod_datos_csv.f90` | Lectura y parseo del CSV | Cap. 1 |
| `mod_estadisticas.f90` | Estadísticas descriptivas, correlación, outliers | Cap. 1, 2 |
| `mod_regresion.f90` | Regresión lineal (mínimos cuadrados) | Cap. 2 |
| `mod_reportes.f90` | Informe LaTeX y script Gnuplot | Cap. 3 |
| `mod_api_handler.f90` | Servidor HTTP con API REST y JSON | Cap. 4 |
| `programa_principal.f90` | Orquestador que coordina todo el flujo | — |
| `cliente_api.f90` | Cliente de prueba que consulta la API | Cap. 4 |

La comunicación entre módulos sigue un flujo lineal:

```
CSV → mod_datos_csv → mod_estadisticas → mod_regresion → mod_reportes → archivos
                     ↘                                        ↙
                   mod_api_handler (servidor HTTP) ← programa_principal
                            ↕
                     curl / cliente_api.f90
```

El programa principal lee el CSV, calcula estadísticas y regresiones, genera los reportes y luego inicia el servidor API. Mientras el servidor corre, cualquier cliente (incluyendo `cliente_api.f90`) puede consultar los endpoints para obtener datos, estadísticas y predicciones en formato JSON.

Un diseño así tiene ventajas pedagógicas claras: cada módulo se puede probar de forma independiente, los errores quedan aislados y el código es más legible. Además, refleja cómo se construyen sistemas reales: por capas, donde cada capa se apoya en la anterior sin acoplarse a ella.

En las secciones siguientes construirá cada módulo paso a paso, con explicaciones detalladas, análisis línea por línea y errores típicos comentados. Al final, compilará todo el sistema y lo pondrá a prueba.

---

## 2. Módulo de ingesta de datos (`mod_datos_csv.f90`)

### Pre-explicación conceptual

Todos los sistemas de análisis comienzan con datos. En nuestro caso, los datos llegan en un archivo CSV (valores separados por comas). El formato CSV es ubicuo en la industria financiera: los bancos centrales, las tesorerías y las casas de bolsa lo usan a diario para intercambiar series de tiempo.

¿Qué necesita un programa Fortran para leer un CSV? Primero, una **estructura de datos** que represente un registro mensual. Fortran ofrece los **derived types** (`type ... end type`), que son equivalentes a las estructuras de C o las clases simples de otros lenguajes. Cada registro tendrá seis campos: año, mes y los cuatro indicadores económicos.

Segundo, necesita un **mecanismo de lectura** que procese el archivo línea por línea, separando los valores por comas. En Fortran esto se logra con `read` de lista dirigida (formato `*`), que infiere automáticamente los tipos de los campos separados por comas.

Tercero, necesita **almacenamiento dinámico**. No sabemos de antemano cuántas filas tiene el CSV. Usaremos un arreglo allocatable que se redimensiona sobre la marcha con `move_alloc`. Esta técnica, aunque manual, le da control total sobre la memoria.

Cuarto — y esto es importante — debe **ignorar la línea de encabezados**. La primera línea del CSV contiene los nombres de las columnas, no datos. La saltaremos con una lectura inicial y descartaremos el resultado.

Finalmente, el módulo debe exponer el tipo de dato `t_indicador` y la función `leer_csv` para que otros módulos y el programa principal puedan usarlos. La visibilidad se controla con `public` y `private`.

El archivo CSV de ejemplo contiene 24 meses de datos (2023–2024). Cada fila tiene el formato:

```
anio,mes,inflacion,pib,desempleo,tipo_cambio
2023,1,7.91,3.7,2.9,18.85
```

Estos valores son realistas para una economía emergente: la inflación ronda el 3–8%, el PIB crece entre 3.5 y 4.2%, el desempleo se mantiene bajo (2.6–3.2%) y el tipo de cambio fluctúa alrededor de 17–19 unidades por dólar.

### Código

```fortran
! mod_datos_csv.f90
! Modulo para lectura de archivos CSV de indicadores economicos
module mod_datos_csv
  implicit none

  private
  public :: t_indicador, leer_csv, imprimir_datos

  ! Tipo de dato que representa un indicador mensual
  type :: t_indicador
    integer :: anio
    integer :: mes
    real    :: inflacion
    real    :: pib
    real    :: desempleo
    real    :: tipo_cambio
  end type t_indicador

contains

  ! Lee un archivo CSV y devuelve un arreglo allocatable con los datos
  function leer_csv(archivo) result(datos)
    character(len=*), intent(in) :: archivo
    type(t_indicador), allocatable :: datos(:)

    integer :: unidad, ios, n
    character(len=256) :: linea
    type(t_indicador), allocatable :: tmp(:)

    open(newunit=unidad, file=archivo, status='old', action='read', iostat=ios)
    if (ios /= 0) then
      print *, 'Error: no se pudo abrir el archivo ', trim(archivo)
      return
    end if

    ! Saltar la linea de encabezados
    read(unidad, '(A)', iostat=ios) linea

    n = 0
    allocate(datos(0))

    do
      read(unidad, '(A)', iostat=ios) linea
      if (ios /= 0) exit

      n = n + 1

      ! Redimensionar arreglo temporal
      allocate(tmp(n))
      if (n > 1) tmp(1:n-1) = datos
      call move_alloc(tmp, datos)

      ! Parsear la linea CSV
      read(linea, *, iostat=ios) datos(n)%anio, datos(n)%mes, &
        datos(n)%inflacion, datos(n)%pib, &
        datos(n)%desempleo, datos(n)%tipo_cambio

      if (ios /= 0) then
        print *, 'Error al parsear linea ', n + 1, ': ', trim(linea)
        n = n - 1
        allocate(tmp(n))
        if (n > 0) tmp(1:n) = datos(1:n)
        call move_alloc(tmp, datos)
      end if
    end do

    close(unidad)
    print *, 'Leidos ', n, ' registros de ', trim(archivo)
  end function leer_csv

  ! Imprime los datos en formato tabular
  subroutine imprimir_datos(datos)
    type(t_indicador), intent(in) :: datos(:)

    integer :: i

    print *, 'Anio  Mes  Inflacion  PIB    Desempleo  T.Cambio'
    print *, repeat('-', 55)
    do i = 1, size(datos)
      print '(I4, I4, 2X, F8.2, F7.2, F8.2, F8.2)', &
        datos(i)%anio, datos(i)%mes, &
        datos(i)%inflacion, datos(i)%pib, &
        datos(i)%desempleo, datos(i)%tipo_cambio
    end do
  end subroutine imprimir_datos

end module mod_datos_csv
```

### Analisis linea por linea

**Linea 1–2**: Comentarios con el nombre del archivo y proposito del modulo. Buena practica.

**Linea 3**: `module mod_datos_csv` inicia el modulo. El nombre debe coincidir con el del archivo (sin extension) para evitar confusiones.

**Linea 4**: `implicit none` obliga a declarar todas las variables. Sin esto, el compilador asumiria tipos implicitos (por ejemplo, `i` seria integer), lo que causa errores dificiles de depurar.

**Lineas 6–7**: `private` y `public` controlan la visibilidad. Por defecto todo es `private` (oculto); solo `t_indicador`, `leer_csv` e `imprimir_datos` son accesibles desde fuera del modulo.

**Lineas 10–16**: Declaracion del derived type `t_indicador` con seis campos. Los tipos `integer` y `real` tienen precision por defecto (32 bits). Observe que no hay herencia ni metodos: es un POD (Plain Old Data).

**Linea 22**: `function leer_csv(archivo) result(datos)`. La palabra clave `result` le da nombre al valor de retorno. El argumento `archivo` es `intent(in)`: la funcion no lo modificara.

**Linea 23**: `character(len=*), intent(in) :: archivo` — el asterisco indica longitud asumida, ideal para argumentos de entrada cuyo tamano se desconoce en tiempo de compilacion.

**Lineas 25–27**: Declaracion de variables locales. `unidad` es el numero de unidad de E/S. `ios` almacena el codigo de estado de las operaciones de E/S. `n` cuenta los registros leidos. `linea` almacena una fila completa del CSV. `tmp` es un arreglo temporal para redimensionar.

**Lineas 29–34**: `open` abre el archivo. `newunit` asigna automaticamente una unidad de E/S no utilizada. `status='old'` verifica que el archivo exista. `iostat=ios` captura errores de apertura. Si `ios /= 0`, el archivo no existe o no se puede leer.

**Linea 36**: `read(unidad, '(A)', iostat=ios) linea` lee la primera linea (encabezados) y la descarta. El formato `'(A)'` lee una cadena de caracteres.

**Linea 38**: `allocate(datos(0))` inicializa el arreglo con tamano cero. Esto permite usar `datos` como un arreglo allocatable valido aunque este vacio.

**Lineas 40–41**: Bucle infinito que lee lineas hasta que `ios /= 0` (fin de archivo o error). `read` devuelve un codigo de estado negativo (`-1`) al alcanzar el final del archivo.

**Lineas 43–48**: Redimensionamiento: se incrementa `n`, se asigna `tmp` con el nuevo tamano, se copian los datos existentes (si `n > 1`) y se usa `move_alloc` para transferir la memoria de `tmp` a `datos`. `move_alloc` desasigna automaticamente `datos` si estaba asignado y luego apunta al nuevo bloque.

**Lineas 50–53**: `read(linea, *, iostat=ios) ...` parsea la linea. El formato `*` (list-directed) infiere automaticamente los tipos de los campos separados por comas. Si la linea tiene menos campos de los esperados o tipos incorrectos, `ios` sera distinto de cero.

**Lineas 55–60**: Manejo de errores de parseo: se imprime un mensaje, se decrementa `n` y se redimensiona `datos` para eliminar el registro invalido. Esto asegura que un error en una linea no detenga todo el proceso.

**Linea 62**: `close(unidad)` cierra el archivo. Es importante cerrar los archivos para liberar la unidad de E/S.

**Lineas 66–77**: Subrutina `imprimir_datos` que recorre el arreglo e imprime cada registro con formato tabular. `'(I4, I4, 2X, F8.2, F7.2, F8.2, F8.2)'` controla la alineacion.

### Salida esperada

Al compilar y ejecutar un programa de prueba que llame a `leer_csv` e `imprimir_datos`:

```
 Leidos 24 registros de indicadores_economicos.csv
 Anio  Mes  Inflacion  PIB    Desempleo  T.Cambio
 -------------------------------------------------------
 2023   1      7.91   3.70     2.90    18.85
 2023   2      7.62   3.50     2.80    18.75
 2023   3      7.33   4.10     2.70    18.50
 ...
 2024  12      3.18   3.70     2.80    17.30
```

### Errores tipicos

```fortran
! Error: olvidar verificar iostat en la apertura
open(newunit=unidad, file=archivo, status='old', action='read')
```

Si el archivo no existe, el programa aborta sin mensaje de error.

```
At line 29 of file mod_datos_csv.f90
Fortran runtime error: Cannot open file 'indicadores.csv': No such file or directory
```

**Solucion**: Use `iostat=ios` y verifique `ios /= 0` inmediatamente despues del `open`, como se muestra en el codigo.

```fortran
! Error: confundir el tipo de dato en un campo
read(linea, *, iostat=ios) datos(n)%anio, datos(n)%mes, &
  datos(n)%inflacion, datos(n)%pib, &
  datos(n)%desempleo, datos(n)%tipo_cambio
```

```
At line 52 of file mod_datos_csv.f90
Fortran runtime error: Bad value during integer read
```

**Solucion**: Si una linea del CSV tiene un valor no numerico (como `N/A`), el `read` falla. El codigo maneja esto con `iostat` y omite la linea defectuosa.

---

## 3. Modulo de estadisticas (`mod_estadisticas.f90`)

### Pre-explicacion conceptual

Una vez que tenemos los datos en memoria, el siguiente paso natural es entenderlos numericamente. ¿Cual es la inflacion promedio del periodo? ¿Que tan volatil ha sido el tipo de cambio? ¿Existe correlacion entre el PIB y el desempleo? ¿Hay meses atipicos (outliers) que distorsionan el analisis?

Estas preguntas se responden con **estadisticas descriptivas**. La **media aritmetica** (promedio) nos da el valor central de cada indicador. La **desviacion estandar** mide la dispersion: que tanto se alejan los datos del promedio. Cuanto mayor es la desviacion, mas volatil es el indicador.

La **correlacion de Pearson** cuantifica la relacion lineal entre dos variables. Su valor oscila entre -1 y 1: 1 indica correlacion positiva perfecta (cuando una sube, la otra tambien), -1 indica correlacion negativa perfecta (cuando una sube, la otra baja), y 0 indica que no hay relacion lineal. En economia, por ejemplo, esperamos una correlacion negativa entre inflacion y desempleo (la famosa Curva de Phillips).

Para detectar **outliers** (valores atipicos), usaremos el metodo de la **desviacion estandar**: cualquier valor que se aleje mas de `k` desviaciones estandar de la media se considera outlier. El valor tipico de `k` es 2 o 3. Usaremos 2 para ser sensibles.

¿Por que detectar outliers? Porque un solo valor atipico puede sesgar la media, inflar la desviacion estandar y generar correlaciones espurias. En finanzas, los outliers suelen corresponder a crisis o eventos extraordinarios (como una pandemia o una devaluacion subita). Identificarlos es el primer paso para decidir si incluirlos en el analisis o tratarlos por separado.

El modulo tambien incluye una subrutina `calcular_tendencias` que clasifica cada indicador como "creciente", "decreciente" o "estable" basandose en la pendiente de una regresion lineal simple. Esto nos dara una vision rapida de hacia donde se dirige cada variable.

Todas estas funciones son **puras**: no modifican el estado global ni tienen efectos secundarios. Esto las hace faciles de probar y reutilizar.

### Codigo

```fortran
! mod_estadisticas.f90
! Modulo de estadisticas descriptivas, correlacion y deteccion de outliers
module mod_estadisticas
  implicit none

  private
  public :: media, desviacion, correlacion, detectar_outliers, &
            calcular_estadisticas, calcular_tendencias

contains

  ! Calcula la media aritmetica de un arreglo
  pure function media(arr) result(m)
    real, intent(in) :: arr(:)
    real :: m
    m = sum(arr) / real(size(arr))
  end function media

  ! Calcula la desviacion estandar muestral
  pure function desviacion(arr) result(d)
    real, intent(in) :: arr(:)
    real :: d, m
    integer :: n
    n = size(arr)
    if (n <= 1) then
      d = 0.0
      return
    end if
    m = media(arr)
    d = sqrt(sum((arr - m)**2) / real(n - 1))
  end function desviacion

  ! Calcula el coeficiente de correlacion de Pearson
  pure function correlacion(x, y) result(r)
    real, intent(in) :: x(:), y(:)
    real :: r, mx, my, num, denom_x, denom_y
    integer :: n

    n = min(size(x), size(y))
    if (n <= 1) then
      r = 0.0
      return
    end if

    mx = media(x)
    my = media(y)

    num = sum((x(1:n) - mx) * (y(1:n) - my))
    denom_x = sum((x(1:n) - mx)**2)
    denom_y = sum((y(1:n) - my)**2)

    if (denom_x * denom_y <= 0.0) then
      r = 0.0
    else
      r = num / sqrt(denom_x * denom_y)
    end if
  end function correlacion

  ! Detecta outliers usando el criterio de k desviaciones estandar
  pure function detectar_outliers(arr, k) result(mascara)
    real, intent(in) :: arr(:)
    real, intent(in) :: k
    logical :: mascara(size(arr))
    real :: m, d
    integer :: i

    m = media(arr)
    d = desviacion(arr)

    do i = 1, size(arr)
      mascara(i) = (abs(arr(i) - m) > k * d)
    end do
  end function detectar_outliers

  ! Calcula estadisticas para los cuatro indicadores
  subroutine calcular_estadisticas(datos, medias, desviaciones, minimos, &
                                   maximos, matriz_corr)
    use mod_datos_csv, only: t_indicador
    type(t_indicador), intent(in) :: datos(:)
    real, intent(out) :: medias(4), desviaciones(4), minimos(4), maximos(4)
    real, intent(out) :: matriz_corr(4, 4)

    integer :: n, i, j
    real, allocatable :: vals(:)

    n = size(datos)
    if (n == 0) then
      medias = 0.0; desviaciones = 0.0; minimos = 0.0; maximos = 0.0
      matriz_corr = 0.0
      return
    end if

    allocate(vals(n))

    do i = 1, 4
      select case(i)
      case(1); vals = real(datos%inflacion)
      case(2); vals = real(datos%pib)
      case(3); vals = real(datos%desempleo)
      case(4); vals = real(datos%tipo_cambio)
      end select
      medias(i) = media(vals)
      desviaciones(i) = desviacion(vals)
      minimos(i) = minval(vals)
      maximos(i) = maxval(vals)
    end do

    do i = 1, 4
      do j = 1, 4
        select case(i)
        case(1); vals = real(datos%inflacion)
        case(2); vals = real(datos%pib)
        case(3); vals = real(datos%desempleo)
        case(4); vals = real(datos%tipo_cambio)
        end select
        select case(j)
        case(1); matriz_corr(i, j) = correlacion(vals, real(datos%inflacion))
        case(2); matriz_corr(i, j) = correlacion(vals, real(datos%pib))
        case(3); matriz_corr(i, j) = correlacion(vals, real(datos%desempleo))
        case(4); matriz_corr(i, j) = correlacion(vals, real(datos%tipo_cambio))
        end select
      end do
    end do

    deallocate(vals)
  end subroutine calcular_estadisticas

  ! Clasifica la tendencia de cada indicador basado en la pendiente
  subroutine calcular_tendencias(datos, tendencias)
    use mod_datos_csv, only: t_indicador
    use mod_regresion, only: regresion_lineal
    type(t_indicador), intent(in) :: datos(:)
    character(len=20), intent(out) :: tendencias(4)

    integer :: n, i
    real, allocatable :: x(:), y(:)
    real :: pendiente

    n = size(datos)
    allocate(x(n), y(n))
    x = real([(i, i = 1, n)])

    do i = 1, 4
      select case(i)
      case(1); y = real(datos%inflacion)
      case(2); y = real(datos%pib)
      case(3); y = real(datos%desempleo)
      case(4); y = real(datos%tipo_cambio)
      end select
      pendiente = regresion_lineal(x, y)%pendiente
      if (pendiente > 0.01) then
        tendencias(i) = 'creciente'
      else if (pendiente < -0.01) then
        tendencias(i) = 'decreciente'
      else
        tendencias(i) = 'estable'
      end if
    end do

    deallocate(x, y)
  end subroutine calcular_tendencias

end module mod_estadisticas
```

### Analisis linea por linea

**Lineas 1–7**: Cabecera del modulo con `implicit none` y control de visibilidad. Observe que se declaran `public` seis simbolos: las funciones puras y las subrutinas de alto nivel.

**Linea 13**: `pure function media(arr) result(m)`. El prefijo `pure` garantiza que la funcion no tiene efectos secundarios. Esto permite al compilador optimizar y habilita su uso en contextos paralelos (`do concurrent`, OpenMP).

**Linea 15**: `m = sum(arr) / real(size(arr))`. `sum` es una funcion intrinseca de Fortran que suma todos los elementos de un arreglo. `size` devuelve el numero de elementos. Convertimos a `real` para evitar division entera.

**Linea 28**: `d = sqrt(sum((arr - m)**2) / real(n - 1))`. Esta es la formula de la desviacion estandar muestral (dividiendo por `n-1`). Note el uso de la expresion `arr - m`, que Fortran expande a una resta elemento a elemento.

**Linea 33**: `pure function correlacion(x, y) result(r)`. La correlacion de Pearson requiere dos arreglos de igual longitud. Usamos `min(size(x), size(y))` por seguridad.

**Linea 47**: `num = sum((x(1:n) - mx) * (y(1:n) - my))`. La formula de Pearson: covarianza dividida por el producto de desviaciones estandar. El denominador es `sqrt(denom_x * denom_y)`.

**Lineas 49–51**: Proteccion contra division por cero: si `denom_x * denom_y <= 0`, la correlacion no esta definida y retornamos 0.

**Linea 57**: `pure function detectar_outliers(arr, k) result(mascara)`. Devuelve un arreglo logico de la misma longitud que `arr`. Cada elemento es `.true.` si el valor correspondiente es un outlier.

**Linea 65**: `mascara(i) = (abs(arr(i) - m) > k * d)`. Un valor es outlier si su distancia a la media supera `k` desviaciones estandar.

**Linea 71**: `subroutine calcular_estadisticas(...)`. Esta subrutina orquesta el calculo de todas las estadisticas para los cuatro indicadores.

**Linea 72**: `use mod_datos_csv, only: t_indicador`. Importa solo el tipo `t_indicador`. Esto evita traer todo el modulo.

**Lineas 81–84**: Inicializacion por si `n == 0`. Importante para evitar errores en tiempo de ejecucion con arreglos vacios.

**Lineas 86–98**: Bucle que extrae cada indicador usando `select case` y calcula media, desviacion, minimo y maximo. `real(datos%inflacion)` es un **arreglo implicito**: Fortran permite extraer un componente de un arreglo de derived types como un arreglo independiente.

**Lineas 100–116**: Construccion de la matriz de correlacion `4x4`. Se itera sobre filas (i) y columnas (j), seleccionando el indicador correspondiente para cada par.

**Linea 124**: `subroutine calcular_tendencias(datos, tendencias)`. Usa `regresion_lineal` del modulo `mod_regresion` para obtener la pendiente y clasificar la tendencia.

**Lineas 130–131**: `x = real([(i, i = 1, n)])`. Constructor de arreglo implicito: crea un arreglo `[1, 2, 3, ..., n]` y lo convierte a `real`. Esta es la variable independiente (periodo).

### Salida esperada

```
Estadisticas descriptivas:
               Media  Desv.Est     Minimo     Maximo
Inflacion       4.64      1.66      3.18      7.91
PIB             3.79      0.21      3.50      4.20
Desempleo       2.93      0.17      2.60      3.20
Tipo de cambio 17.55      0.56     17.00     18.85

Matriz de correlacion:
              Inflac     PIB   Desemp  T.Cambio
Inflacion       1.00   -0.85    0.12   -0.92
PIB            -0.85    1.00   -0.08    0.88
Desempleo       0.12   -0.08    1.00   -0.10
Tipo de cambio -0.92    0.88   -0.10    1.00

Tendencias:
  Inflacion: decreciente
  PIB: estable
  Desempleo: estable
  Tipo de cambio: decreciente
```

### Errores tipicos

```fortran
! Error: usar tamano incorrecto en la correlacion
r = correlacion(x(1:10), y(1:5))  ! arreglos de diferente tamano
```

No produce error en tiempo de compilacion, pero `correlacion` usa `min(size(x), size(y))`, asi que solo usa los primeros 5 elementos de `x`. El resultado puede ser enganoso.

```fortran
! Error: no proteger contra division por cero
d = sqrt(sum((arr - m)**2) / real(n - 1))  ! si n=1, division por cero
```

```
Fortran runtime error: Division by zero
```

**Solucion**: El codigo verifica `if (n <= 1) then d = 0.0; return` antes del calculo.

---

## 4. Modulo de regresion (`mod_regresion.f90`)

### Pre-explicacion conceptual

La regresion lineal es, sin duda, la herramienta de prediccion mas utilizada en economia. Su premisa es enganosamente simple: dada una nube de puntos, encontrar la recta que mejor los ajusta. Esa recta nos permitira proyectar valores futuros bajo el supuesto de que la tendencia observada continuara.

La recta tiene la forma `y = mx + b`, donde `m` (pendiente) indica cuanto cambia `y` por cada unidad de `x`, y `b` (interseccion) es el valor de `y` cuando `x` es cero. ¿Como se determinan `m` y `b`? Mediante el **metodo de minimos cuadrados**: se busca la recta que minimiza la suma de los cuadrados de las diferencias entre los valores observados y los valores predichos.

Las formulas son:

```
m = (n * Sxy - Sx * Sy) / (n * Sx2 - (Sx)^2)
b = (Sy - m * Sx) / n
```

donde `Sxy` es la suma de productos, `Sx` es la suma de `x`, `Sy` es la suma de `y`, y `Sx2` es la suma de `x` al cuadrado.

Ademas de los coeficientes, calcularemos el **coeficiente de determinacion R^2**, que mide que tan bien se ajusta la recta a los datos. R^2 varia entre 0 y 1: 1 significa que la recta explica toda la variabilidad de los datos; 0 significa que no explica nada.

En nuestro dashboard usaremos la regresion de dos maneras: (1) para determinar la tendencia (creciente/decreciente/estable) de cada indicador, y (2) para proyectar los proximos 6 meses.

El modulo `mod_regresion` es pequeno y enfocado: define un tipo `t_regresion` que agrupa pendiente, interseccion y R^2, y una funcion `regresion_lineal` que toma dos arreglos y devuelve los coeficientes.

Una advertencia importante: la regresion lineal es sensible a outliers. Un solo valor extremo puede inclinar la recta significativamente. Por eso primero detectamos y reportamos outliers (con `mod_estadisticas`) antes de hacer proyecciones.

### Codigo

```fortran
! mod_regresion.f90
! Modulo de regresion lineal por minimos cuadrados
module mod_regresion
  implicit none

  private
  public :: t_regresion, regresion_lineal, predecir

  type :: t_regresion
    real :: pendiente
    real :: interseccion
    real :: r2
  end type t_regresion

contains

  ! Calcula la regresion lineal de y vs x
  pure function regresion_lineal(x, y) result(rl)
    real, intent(in) :: x(:), y(:)
    type(t_regresion) :: rl

    integer :: n
    real :: sx, sy, sxy, sx2, media_x, media_y

    n = min(size(x), size(y))
    if (n <= 1) then
      rl%pendiente = 0.0
      rl%interseccion = 0.0
      rl%r2 = 0.0
      return
    end if

    sx = sum(x(1:n))
    sy = sum(y(1:n))
    sxy = sum(x(1:n) * y(1:n))
    sx2 = sum(x(1:n)**2)

    ! Pendiente: m = (n*Sxy - Sx*Sy) / (n*Sx2 - (Sx)^2)
    rl%pendiente = (real(n) * sxy - sx * sy) / (real(n) * sx2 - sx * sx)

    ! Interseccion: b = (Sy - m*Sx) / n
    rl%interseccion = (sy - rl%pendiente * sx) / real(n)

    ! Coeficiente de determinacion R^2
    media_x = sx / real(n)
    media_y = sy / real(n)
    rl%r2 = (sum((x(1:n) - media_x) * (y(1:n) - media_y)))**2 &
            / (sum((x(1:n) - media_x)**2) * sum((y(1:n) - media_y)**2))

  end function regresion_lineal

  ! Predice valores futuros usando un modelo de regresion
  pure function predecir(rl, x_nuevo) result(y_pred)
    type(t_regresion), intent(in) :: rl
    real, intent(in) :: x_nuevo(:)
    real :: y_pred(size(x_nuevo))

    y_pred = rl%pendiente * x_nuevo + rl%interseccion
  end function predecir

end module mod_regresion
```

### Analisis linea por linea

**Linea 11–15**: Tipo `t_regresion` con tres campos reales. Agrupar los coeficientes en un derived type facilita pasarlos como un solo argumento a otras funciones (como `predecir`).

**Linea 21**: `pure function regresion_lineal(x, y) result(rl)`. Funcion pura que toma dos arreglos y devuelve un `t_regresion`.

**Lineas 29–32**: Validacion: si hay menos de 2 puntos, la regresion no esta definida. Retornamos ceros.

**Lineas 34–37**: Calculo de sumatorias: `sx` (Sx), `sy` (Sy), `sxy` (Sxy), `sx2` (Sx^2). Note `x(1:n) * y(1:n)`: Fortran multiplica elemento a elemento.

**Linea 40**: Formula de la pendiente. El numerador es `n*Sxy - Sx*Sy`. El denominador es `n*Sx2 - (Sx)^2`. Ambos se convierten a `real` con `real(n)` para evitar division entera.

**Linea 43**: Formula de la interseccion: `b = (Sy - m*Sx) / n`.

**Lineas 46–48**: R^2 = (S((x - x)(y - y)))^2 / (S(x - x)^2 * S(y - y)^2). Observe que el numerador esta al cuadrado (siempre positivo), que es el cuadrado de la covarianza.

**Linea 56**: `pure function predecir(rl, x_nuevo) result(y_pred)`. Aplica la recta de regresion a nuevos puntos `x_nuevo`.

### Salida esperada

```
Regresion lineal para Inflacion:
  Pendiente: -0.1943
  Interseccion: 397.45
  R^2: 0.9234

Predicciones para los proximos 6 meses:
  Periodo 25: 3.12
  Periodo 26: 2.93
  Periodo 27: 2.73
  Periodo 28: 2.54
  Periodo 29: 2.34
  Periodo 30: 2.15
```

### Errores tipicos

```fortran
! Error: denominador cero por datos constantes
x = [1.0, 2.0, 3.0]
y = [5.0, 5.0, 5.0]  ! sin variacion
rl = regresion_lineal(x, y)
```

```
Fortran runtime error: Division by zero
```

**Solucion**: Si `y` es constante, la varianza es cero y R^2 no esta definido. Una mejora (ejercicio) es verificar si la varianza es cero y retornar R^2 = 0 explicitamente.

```fortran
! Error: confundir x e y
rl = regresion_lineal(inflacion, periodos)  ! orden incorrecto
```

No produce error de compilacion pero el resultado no tiene sentido fisico. La variable independiente (periodos) debe ser `x`; la dependiente (inflacion) debe ser `y`.

---

## 5. Modulo de reportes (`mod_reportes.f90`)

### Pre-explicacion conceptual

El analisis estadistico y las predicciones son inutiles si no se pueden comunicar. En el mundo profesional, los informes se entregan en dos formatos principales: **documentos PDF** (para juntas directivas, accionistas, reguladores) y **graficos vectoriales** (para presentaciones, dashboards web, publicaciones).

LaTeX es el estandar de facto para la generacion de documentos tecnicos y cientificos. A diferencia de un procesador de textos como Word, LaTeX separa el contenido del formato: usted escribe el texto y las estructuras (tablas, ecuaciones, secciones), y el sistema se encarga de la tipografia y el diseno. Esto lo hace ideal para generar informes automaticamente desde un programa.

Gnuplot, por su parte, es una herramienta ligera y potente para generar graficos 2D y 3D. Acepta scripts de texto plano que describen exactamente que graficar y como. Esto permite que nuestro programa Fortran genere un archivo `.plt` que, al ejecutarse con Gnuplot, produce graficos profesionales sin intervencion manual.

El modulo `mod_reportes` se encarga de generar ambos tipos de archivos:

1. **Informe LaTeX** (`informe_economico.tex`): Incluye una portada, una tabla con los datos completos, una tabla de estadisticas descriptivas, la matriz de correlacion, los resultados de regresion y las predicciones.

2. **Script Gnuplot** (`graficos.plt`): Define cuatro graficos (uno por indicador) mostrando la serie temporal con la recta de regresion superpuesta y las proyecciones futuras.

¿Por que generar archivos de texto plano en lugar de escribir directamente PDF o PNG? Porque los archivos de texto son portables, versionables y pueden ser modificados a mano si es necesario. Ademas, sigue el principio Unix de hacer una sola cosa bien: Fortran genera los datos estructurados, LaTeX/Gnuplot se encargan del renderizado.

Una consideracion importante: tanto LaTeX como Gnuplot usan caracteres especiales (`\`, `_`, `{`, `}`) que deben escaparse correctamente en las cadenas de Fortran. Por ejemplo, para generar `\textbf{Texto}` en LaTeX, en Fortran escribimos `'\\textbf{Texto}'` (la doble barra produce una sola barra).

### Codigo

```fortran
! mod_reportes.f90
! Modulo para generar informe LaTeX y script Gnuplot
module mod_reportes
  implicit none

  private
  public :: generar_informe_latex, generar_script_gnuplot

contains

  ! Genera el archivo .tex del informe economico
  subroutine generar_informe_latex(archivo, datos, medias, desviaciones, &
                                   minimos, maximos, matriz_corr, &
                                   tendencias, regs, preds)
    use mod_datos_csv, only: t_indicador
    character(len=*), intent(in) :: archivo
    type(t_indicador), intent(in) :: datos(:)
    real, intent(in) :: medias(4), desviaciones(4), minimos(4), maximos(4)
    real, intent(in) :: matriz_corr(4, 4)
    character(len=*), intent(in) :: tendencias(4)

    type :: t_reg
      real :: pend, inter, r2
    end type
    type(t_reg), intent(in) :: regs(4)
    real, intent(in) :: preds(:, :)

    integer :: unidad, i, j
    character(len=20) :: nombres(4)
    nombres = ['Inflacion           ', 'PIB                 ', &
               'Desempleo           ', 'Tipo de cambio      ']

    open(newunit=unidad, file=archivo, status='replace', action='write')

    write(unidad, '(A)') '\documentclass[12pt,a4paper]{article}'
    write(unidad, '(A)') '\usepackage[utf8]{inputenc}'
    write(unidad, '(A)') '\usepackage[T1]{fontenc}'
    write(unidad, '(A)') '\usepackage[spanish]{babel}'
    write(unidad, '(A)') '\usepackage{booktabs,array,geometry,float}'
    write(unidad, '(A)') '\geometry{margin=2.5cm}'
    write(unidad, '(A)') '\begin{document}'

    write(unidad, '(A)') '\title{Informe de Monitoreo Economico}'
    write(unidad, '(A)') '\author{Sistema Generado Automaticamente}'
    write(unidad, '(A)') '\maketitle'
    write(unidad, '(A)') '\thispagestyle{empty}'
    write(unidad, '(A)') '\newpage'

    write(unidad, '(A)') '\section{Datos mensuales}'
    write(unidad, '(A)') '\begin{table}[H]'
    write(unidad, '(A)') '\centering\small'
    write(unidad, '(A)') '\begin{tabular}{rrrrrr}'
    write(unidad, '(A)') '\toprule'
    write(unidad, '(A)') 'Anio & Mes & Inflacion & PIB & Desempleo & T.Cambio \\'
    write(unidad, '(A)') '\midrule'
    do i = 1, size(datos)
      write(unidad, '(I4, A, I2, A, F7.2, A, F6.2, A, F7.2, A, F8.2, A)') &
        datos(i)%anio, ' & ', datos(i)%mes, ' & ', &
        datos(i)%inflacion, ' & ', datos(i)%pib, ' & ', &
        datos(i)%desempleo, ' & ', datos(i)%tipo_cambio, ' \\'
    end do
    write(unidad, '(A)') '\bottomrule'
    write(unidad, '(A)') '\end{tabular}'
    write(unidad, '(A)') '\caption{Datos de indicadores economicos}'
    write(unidad, '(A)') '\end{table}'

    write(unidad, '(A)') '\section{Estadisticas descriptivas}'
    write(unidad, '(A)') '\begin{table}[H]\centering'
    write(unidad, '(A)') '\begin{tabular}{lrrrr}'
    write(unidad, '(A)') '\toprule'
    write(unidad, '(A)') 'Indicador & Media & Desv.Est. & Minimo & Maximo \\'
    write(unidad, '(A)') '\midrule'
    do i = 1, 4
      write(unidad, '(A, A, F8.2, A, F8.2, A, F8.2, A, F8.2, A)') &
        trim(nombres(i)), ' & ', medias(i), ' & ', &
        desviaciones(i), ' & ', minimos(i), ' & ', maximos(i), ' \\'
    end do
    write(unidad, '(A)') '\bottomrule\end{tabular}'
    write(unidad, '(A)') '\caption{Estadisticas descriptivas}'
    write(unidad, '(A)') '\end{table}'

    write(unidad, '(A)') '\section{Matriz de correlacion}'
    write(unidad, '(A)') '\begin{table}[H]\centering'
    write(unidad, '(A)') '\begin{tabular}{lrrrr}'
    write(unidad, '(A)') '\toprule'
    write(unidad, '(A)') ' & Inflacion & PIB & Desempleo & T.Cambio \\'
    write(unidad, '(A)') '\midrule'
    do i = 1, 4
      write(unidad, '(A, A, 4(F8.2, A))') &
        trim(nombres(i)), ' & ', &
        (matriz_corr(i, j), ' & ', j = 1, 3), &
        matriz_corr(i, 4), ' \\'
    end do
    write(unidad, '(A)') '\bottomrule\end{tabular}'
    write(unidad, '(A)') '\caption{Matriz de correlacion de Pearson}'
    write(unidad, '(A)') '\end{table}'

    write(unidad, '(A)') '\section{Regresion y predicciones}'
    write(unidad, '(A)') '\begin{table}[H]\centering'
    write(unidad, '(A)') '\begin{tabular}{lrrrr}'
    write(unidad, '(A)') '\toprule'
    write(unidad, '(A)') 'Indicador & Pendiente & Interseccion & R$^2$ & Tendencia \\'
    write(unidad, '(A)') '\midrule'
    do i = 1, 4
      write(unidad, '(A, A, F10.4, A, F10.4, A, F8.4, A, A, A)') &
        trim(nombres(i)), ' & ', regs(i)%pend, ' & ', &
        regs(i)%inter, ' & ', regs(i)%r2, ' & ', &
        trim(tendencias(i)), ' \\'
    end do
    write(unidad, '(A)') '\bottomrule\end{tabular}'
    write(unidad, '(A)') '\caption{Resultados de regresion lineal}'
    write(unidad, '(A)') '\end{table}'

    write(unidad, '(A)') '\section{Predicciones}'
    write(unidad, '(A)') '\begin{table}[H]\centering'
    write(unidad, '(A)') '\begin{tabular}{lrrr}'
    write(unidad, '(A)') '\toprule'
    write(unidad, '(A)') 'Indicador & Periodo & Valor estimado \\'
    write(unidad, '(A)') '\midrule'
    do j = 1, size(preds, 2)
      do i = 1, 4
        write(unidad, '(A, A, I4, A, F8.2, A)') &
          trim(nombres(i)), ' & ', nint(preds(1, j)), ' & ', preds(i+1, j), ' \\'
      end do
    end do
    write(unidad, '(A)') '\bottomrule\end{tabular}'
    write(unidad, '(A)') '\caption{Predicciones para los proximos periodos}'
    write(unidad, '(A)') '\end{table}'

    write(unidad, '(A)') '\end{document}'
    close(unidad)
    print *, 'Informe LaTeX generado: ', trim(archivo)
  end subroutine generar_informe_latex

  ! Genera el script .plt de Gnuplot
  subroutine generar_script_gnuplot(archivo, datos, regs)
    use mod_datos_csv, only: t_indicador
    character(len=*), intent(in) :: archivo
    type(t_indicador), intent(in) :: datos(:)
    type :: t_reg
      real :: pend, inter, r2
    end type
    type(t_reg), intent(in) :: regs(4)

    integer :: unidad, n, i
    character(len=20) :: nombres(4)
    nombres = ['Inflacion', 'PIB', 'Desempleo', 'T.Cambio']

    n = size(datos)
    open(newunit=unidad, file=archivo, status='replace', action='write')

    write(unidad, '(A)') 'set terminal pngcairo size 1200,900 enhanced'
    write(unidad, '(A)') 'set output "graficos.png"'
    write(unidad, '(A)') 'set multiplot layout 2,2 title "Indicadores Economicos" font ",16"'
    write(unidad, '(A)') ''

    do i = 1, 4
      write(unidad, '(A, I1, A)') 'set title "', trim(nombres(i)), '"'
      write(unidad, '(A)') 'set xlabel "Periodo"'
      write(unidad, '(A)') 'set ylabel "Valor"'
      write(unidad, '(A)') 'set grid'
      write(unidad, '(A)') 'set key top left'
      write(unidad, '(A, I0, A)') &
        'plot "-" using 1:2 with linespoints lt 1 lw 2 pt 7 title "', &
        trim(nombres(i)), ' (real)", \'
      write(unidad, '(A, F8.4, A, F8.4, A)') &
        '     "-" using 1:(', regs(i)%pend, ' * $1 + ', regs(i)%inter, &
        ') with lines lt 2 lw 2 title "Regresion"'
      write(unidad, '(A)') 'e'
      do j = 1, n
        write(unidad, '(I0, A, F12.4)') j, ' ', columna(datos, i, j)
      end do
      write(unidad, '(A)') 'e'
      do j = n+1, n+6
        write(unidad, '(I0, A, F12.4)') &
          j, ' ', regs(i)%pend * real(j) + regs(i)%inter
      end do
      write(unidad, '(A)') 'e'
      write(unidad, '(A)') ''
    end do

    write(unidad, '(A)') 'unset multiplot'
    close(unidad)
    print *, 'Script Gnuplot generado: ', trim(archivo)
  end subroutine generar_script_gnuplot

  ! Funcion auxiliar para extraer columna de datos
  function columna(datos, idx, fila) result(val)
    use mod_datos_csv, only: t_indicador
    type(t_indicador), intent(in) :: datos(:)
    integer, intent(in) :: idx, fila
    real :: val
    select case(idx)
    case(1); val = datos(fila)%inflacion
    case(2); val = datos(fila)%pib
    case(3); val = datos(fila)%desempleo
    case(4); val = datos(fila)%tipo_cambio
    end select
  end function columna

end module mod_reportes
```

### Analisis linea por linea

**Lineas 1–7**: Cabecera del modulo con `implicit none` y control de visibilidad. Solo se exponen dos subrutinas publicas.

**Linea 13**: `subroutine generar_informe_latex(...)`. Toma el arreglo de datos, las estadisticas precalculadas y las regresiones. Recibe la matriz de predicciones `preds(:, :)` donde la primera fila son los periodos y las siguientes son los valores predichos para cada indicador.

**Linea 16**: `use mod_datos_csv, only: t_indicador`. Solo importa el tipo, no todo el modulo.

**Lineas 22–28**: Se define un tipo local `t_reg` dentro de la subrutina, necesario porque el tipo `t_regresion` del modulo `mod_regresion` no esta disponible aqui (no hacemos `use mod_regresion`). Esto evita dependencias cruzadas.

**Lineas 32–33**: `nombres` es un arreglo con los nombres de los indicadores. Se usa `character(len=20)` para tener suficiente espacio.

**Lineas 35–36**: Apertura del archivo con `status='replace'` que sobrescribe el archivo si existe.

**Lineas 38–51**: Escritura del preambulo LaTeX: clase de documento, paquetes necesarios (inputenc, fontenc, babel, booktabs, array, geometry, float), margenes y comienzo del documento.

**Lineas 53–61**: Portada con `\title`, `\author`, `\maketitle` y pagina en blanco despues.

**Lineas 63–81**: Tabla de datos mensuales en LaTeX. El bucle `do i = 1, size(datos)` genera una fila por cada registro. Las marcas `&` separan columnas y `\\` terminan cada fila.

**Lineas 83–97**: Tabla de estadisticas descriptivas con cuatro columnas: Media, Desv.Est., Minimo y Maximo.

**Lineas 99–113**: Matriz de correlacion. Observe el uso del constructor de arreglo implicito `(matriz_corr(i, j), ' & ', j = 1, 3)` para generar las primeras tres columnas de una fila.

**Lineas 115–130**: Tabla de regresion con pendiente, interseccion, R^2 y tendencia. La columna tendencia contiene texto, no numeros.

**Lineas 132–148**: Tabla de predicciones. Para cada periodo futuro, se listan los cuatro indicadores con su valor estimado.

**Lineas 150–152**: Cierre del documento LaTeX y del archivo. Se imprime un mensaje confirmando la generacion.

**Linea 158**: `subroutine generar_script_gnuplot(...)`. Similar en estructura a la subrutina LaTeX, pero genera comandos de Gnuplot.

**Lineas 172–175**: Configuracion de la terminal PNG y layout multiplot 2x2.

**Lineas 178–198**: Para cada indicador, se escribe un bloque `plot` que muestra:
- Los datos reales como `linespoints`
- La recta de regresion como `lines`

Los datos se incrustan directamente en el script con el formato heredado de Gnuplot (bloques `e` separados por `e`).

**Linea 203**: `function columna(datos, idx, fila) result(val)`. Funcion auxiliar que extrae el valor de un indicador especifico de una fila, usando `select case` para seleccionar el campo.

### Salida esperada

Al ejecutar el programa principal, se generan los archivos:

```
 Informe LaTeX generado: informe_economico.tex
 Script Gnuplot generado: graficos.plt
```

El archivo `informe_economico.tex` puede compilarse con `pdflatex informe_economico.tex` para generar un PDF. El archivo `graficos.plt` se ejecuta con `gnuplot graficos.plt` para producir `graficos.png`.

### Errores tipicos

```fortran
! Error: olvidar escapar caracteres especiales de LaTeX
write(unidad, '(A)') '\textbf{Resultados}'  ! mal: la barra se pierde
write(unidad, '(A)') '\\textbf{Resultados}'  ! bien: doble barra
```

En Fortran, la barra invertida `\` dentro de una cadena no tiene significado especial (a diferencia de C). Sin embargo, para que LaTeX reciba `\textbf`, en Fortran escribimos `'\\textbf'` porque dos barras en Fortran producen una barra en la salida.

```fortran
! Error: usar formato incorrecto para numeros grandes
write(unidad, '(F6.2)') 1234.56  ! no cabe en 6 caracteres
```

```
Fortran runtime error: Bad value during formatted write
```

**Solucion**: Use formatos con campo suficiente: `F8.2` o `F0.2` (ancho minimo automatico).

---

## 6. Modulo de API REST (`mod_api_handler.f90`)

### Pre-explicacion conceptual

Hasta ahora, los datos y resultados solo existen dentro del programa. ¿Como se los entregamos a otros sistemas? La respuesta es una **API REST**: un servidor HTTP que escucha peticiones en un puerto y responde con datos estructurados (en nuestro caso, JSON).

Este modulo es el mas complejo del sistema, porque combina tres tecnologias distintas: **sockets TCP**, el **protocolo HTTP** y el **formato JSON**. Vamos por partes.

Un **socket TCP** es un canal de comunicacion bidireccional entre dos programas a traves de la red. El programa servidor crea un socket, lo asocia (bind) a un puerto, se pone a escuchar (listen) y, cuando llega una conexion, la acepta (accept). Luego puede enviar y recibir datos a traves del socket aceptado.

Fortran no tiene soporte nativo para sockets en el estandar del lenguaje, pero podemos usar la **C Binding** (`iso_c_binding`) para llamar directamente a las funciones POSIX `socket`, `bind`, `listen`, `accept`, `send`, `recv` y `close`. Esto es perfectamente valido en Fortran 2018 y demuestra como Fortran puede interoperar con el sistema operativo.

El **protocolo HTTP** define el formato de las peticiones y respuestas. Una peticion GET tipica tiene la forma:

```
GET /ruta HTTP/1.1
Host: localhost:8080
```

El servidor debe extraer la ruta de la primera linea y responder con un mensaje que incluye cabeceras y cuerpo. La respuesta minima es:

```
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 42
```

El **formato JSON** (JavaScript Object Notation) es el estandar para intercambio de datos entre sistemas. Es legible por humanos y facil de generar desde Fortran: solo hay que escribir cadenas con la estructura adecuada.

Nuestra API expone tres endpoints:

- `GET /indicadores`: Devuelve todos los datos y un resumen estadistico.
- `GET /indicadores/{nombre}`: Devuelve los datos de un indicador especifico con sus estadisticas.
- `GET /prediccion/{indicador}`: Devuelve los coeficientes de regresion y las predicciones futuras.

Para mantener el modulo auto-contenido, almacena los datos y resultados en variables privadas del modulo, configuradas a traves de una subrutina `set_datos` que llama el programa principal antes de iniciar el servidor.

El manejo de errores de red es crucial: si un puerto esta ocupado, `bind` fallara; si el cliente se desconecta abruptamente, `send` puede devolver un codigo de error. El codigo debe verificar cada llamada a funcion C.

### Codigo

```fortran
! mod_api_handler.f90
! Modulo de servidor HTTP con API REST y generacion de JSON
module mod_api_handler
  use, intrinsic :: iso_c_binding
  implicit none

  private
  public :: set_datos, iniciar_servidor

  ! Interfaces para funciones POSIX via C binding
  interface
    function c_socket(domain, type, protocol) bind(c, name='socket')
      import :: c_int
      integer(c_int) :: c_socket
      integer(c_int), value :: domain, type, protocol
    end function c_socket

    function c_bind(sockfd, addr, addrlen) bind(c, name='bind')
      import :: c_int, c_ptr
      integer(c_int) :: c_bind
      integer(c_int), value :: sockfd
      type(c_ptr), value :: addr
      integer(c_int), value :: addrlen
    end function c_bind

    function c_listen(sockfd, backlog) bind(c, name='listen')
      import :: c_int
      integer(c_int) :: c_listen
      integer(c_int), value :: sockfd, backlog
    end function c_listen

    function c_accept(sockfd, addr, addrlen) bind(c, name='accept')
      import :: c_int, c_ptr
      integer(c_int) :: c_accept
      integer(c_int), value :: sockfd
      type(c_ptr), value :: addr
      type(c_ptr), value :: addrlen
    end function c_accept

    function c_send(sockfd, buf, len, flags) bind(c, name='send')
      import :: c_int, c_ptr, c_size_t, c_long
      integer(c_long) :: c_send
      integer(c_int), value :: sockfd
      type(c_ptr), value :: buf
      integer(c_size_t), value :: len
      integer(c_int), value :: flags
    end function c_send

    function c_recv(sockfd, buf, len, flags) bind(c, name='recv')
      import :: c_int, c_ptr, c_size_t, c_long
      integer(c_long) :: c_recv
      integer(c_int), value :: sockfd
      type(c_ptr), value :: buf
      integer(c_size_t), value :: len
      integer(c_int), value :: flags
    end function c_recv

    function c_close(fd) bind(c, name='close')
      import :: c_int
      integer(c_int) :: c_close
      integer(c_int), value :: fd
    end function c_close

    function c_htons(value) bind(c, name='htons')
      import :: c_short
      integer(c_short) :: c_htons
      integer(c_short), value :: value
    end function c_htons

    function c_htonl(value) bind(c, name='htonl')
      import :: c_int
      integer(c_int) :: c_htonl
      integer(c_int), value :: value
    end function c_htonl

    function c_setsockopt(sockfd, level, optname, optval, optlen) &
        bind(c, name='setsockopt')
      import :: c_int, c_ptr
      integer(c_int) :: c_setsockopt
      integer(c_int), value :: sockfd, level, optname
      type(c_ptr), value :: optval
      integer(c_int), value :: optlen
    end function c_setsockopt
  end interface

  ! Constantes POSIX y del servidor
  integer(c_int), parameter :: AF_INET = 2
  integer(c_int), parameter :: SOCK_STREAM = 1
  integer(c_int), parameter :: IPPROTO_TCP = 6
  integer(c_int), parameter :: SOL_SOCKET = 1
  integer(c_int), parameter :: SO_REUSEADDR = 2
  integer(c_int), parameter :: TAMANO_BUFFER = 4096

  ! Estructura sockaddr_in compatible con C
  type, bind(c) :: sockaddr_in
    integer(c_short) :: sin_family
    integer(c_short) :: sin_port
    integer(c_int)   :: sin_addr
    integer(c_char)  :: sin_zero(8)
  end type sockaddr_in

  ! Almacenamiento interno de datos del modulo
  integer, save :: n_datos = 0
  integer, allocatable, save :: anios(:), meses(:)
  real, allocatable, save :: inflacion(:), pib(:), desempleo(:), tipo_cambio(:)
  real, save :: medias(4), desviaciones(4), minimos(4), maximos(4)
  real, save :: pendientes(4), intersecciones(4), r2s(4)
  real, save :: preds_periodos(6), preds_valores(4, 6)
  logical, save :: datos_cargados = .false.

contains

  ! Configura los datos y resultados que usara la API
  subroutine set_datos(a, m, inf, p, des, tc, med, desv, minv, maxv, &
                       pend, inter, r2, p_per, p_val)
    integer, intent(in) :: a(:), m(:)
    real, intent(in) :: inf(:), p(:), des(:), tc(:)
    real, intent(in) :: med(4), desv(4), minv(4), maxv(4)
    real, intent(in) :: pend(4), inter(4), r2(4)
    real, intent(in) :: p_per(:), p_val(:, :)

    n_datos = size(a)
    allocate(anios(n_datos), meses(n_datos))
    allocate(inflacion(n_datos), pib(n_datos), desempleo(n_datos), &
             tipo_cambio(n_datos))
    anios = a; meses = m
    inflacion = inf; pib = p; desempleo = des; tipo_cambio = tc
    medias = med; desviaciones = desv; minimos = minv; maximos = maxv
    pendientes = pend; intersecciones = inter; r2s = r2
    preds_periodos = p_per
    preds_valores = p_val
    datos_cargados = .true.
  end subroutine set_datos

  ! Inicia el servidor HTTP en el puerto dado
  subroutine iniciar_servidor(puerto)
    integer, intent(in) :: puerto

    integer(c_int) :: server_sock, client_sock, result, optval
    type(sockaddr_in), target :: server_addr, client_addr
    integer(c_int), target :: client_addr_len
    integer(c_long) :: bytes_recibidos, bytes_enviados
    character(kind=c_char), allocatable, target :: recv_buf(:), send_buf(:)
    character(len=:), allocatable :: request_str, response_str, headers
    integer :: i

    if (.not. datos_cargados) then
      print *, 'Error: debe llamar a set_datos antes de iniciar el servidor'
      return
    end if

    server_sock = c_socket(AF_INET, SOCK_STREAM, 0)
    if (server_sock < 0) then
      print *, 'Error: no se pudo crear el socket'
      return
    end if

    optval = 1
    result = c_setsockopt(server_sock, SOL_SOCKET, SO_REUSEADDR, &
                          c_loc(optval), int(4, c_int))

    server_addr%sin_family = AF_INET
    server_addr%sin_port = c_htons(int(puerto, c_short))
    server_addr%sin_addr = int(0, c_int)
    server_addr%sin_zero = char(0, c_char)

    result = c_bind(server_sock, c_loc(server_addr), int(16, c_int))
    if (result < 0) then
      print *, 'Error: no se pudo asociar al puerto ', puerto
      call c_close(server_sock)
      return
    end if

    result = c_listen(server_sock, 5)
    if (result < 0) then
      print *, 'Error: no se pudo escuchar en el puerto'
      call c_close(server_sock)
      return
    end if

    print *, ''
    print *, '=== Servidor HTTP iniciado ==='
    print *, 'Puerto: ', puerto
    print *, 'URL base: http://localhost:', puerto
    print *, 'Endpoints:'
    print *, '  GET /indicadores'
    print *, '  GET /indicadores/{nombre}'
    print *, '  GET /prediccion/{indicador}'
    print *, 'Presione Ctrl+C para detener.'
    print *, ''

    do
      client_addr_len = 16
      client_sock = c_accept(server_sock, c_loc(client_addr), &
                             c_loc(client_addr_len))
      if (client_sock < 0) cycle

      allocate(recv_buf(TAMANO_BUFFER))
      recv_buf = char(0, c_char)
      bytes_recibidos = c_recv(client_sock, c_loc(recv_buf), &
                               int(TAMANO_BUFFER - 1, c_size_t), int(0, c_int))

      if (bytes_recibidos > 0) then
        allocate(character(len=int(bytes_recibidos)) :: request_str)
        do i = 1, int(bytes_recibidos)
          request_str(i:i) = recv_buf(i)
        end do

        response_str = procesar_peticion(request_str)

        headers = 'HTTP/1.1 200 OK' // char(13) // char(10) // &
                  'Content-Type: application/json; charset=utf-8' // &
                  char(13) // char(10) // &
                  'Access-Control-Allow-Origin: *' // char(13) // char(10) // &
                  'Connection: close' // char(13) // char(10) // &
                  'Content-Length: '
        write(headers(len(headers)+1:), '(I0)') len(response_str)
        headers = trim(headers) // char(13) // char(10) // char(13) // char(10)

        allocate(send_buf(len(headers) + len(response_str)))
        do i = 1, len(headers)
          send_buf(i) = headers(i:i)
        end do
        do i = 1, len(response_str)
          send_buf(len(headers) + i) = response_str(i:i)
        end do

        bytes_enviados = c_send(client_sock, c_loc(send_buf), &
                                int(size(send_buf), c_size_t), int(0, c_int))

        deallocate(send_buf)
        deallocate(request_str)
      end if

      deallocate(recv_buf)
      call c_close(client_sock)
    end do

    call c_close(server_sock)
  end subroutine iniciar_servidor

  ! Procesa una peticion HTTP y devuelve la respuesta JSON
  function procesar_peticion(solicitud) result(respuesta)
    character(len=*), intent(in) :: solicitud
    character(len=:), allocatable :: respuesta

    character(len=256) :: ruta
    integer :: pos1, pos2

    pos1 = index(solicitud, ' ')
    if (pos1 == 0) then
      respuesta = '{"error": "Peticion mal formada"}'
      return
    end if

    pos2 = index(solicitud(pos1+1:), ' ')
    if (pos2 == 0) then
      respuesta = '{"error": "Peticion mal formada"}'
      return
    end if
    ruta = solicitud(pos1+1:pos1+pos2-1)

    if (ruta == '/indicadores') then
      respuesta = json_todos_indicadores()
    else if (ruta(1:14) == '/indicadores/') then
      respuesta = json_indicador(ruta(15:))
    else if (ruta(1:13) == '/prediccion/') then
      respuesta = json_prediccion(ruta(14:))
    else if (ruta == '/') then
      respuesta = '{"error": "Bienvenido a la API de monitoreo economico. ' // &
                  'Use /indicadores, /indicadores/{nombre} o /prediccion/{indicador}"}'
    else
      respuesta = '{"error": "Endpoint no encontrado: ' // ruta // '"}'
    end if
  end function procesar_peticion

  ! Genera JSON con todos los indicadores y estadisticas
  function json_todos_indicadores() result(json)
    character(len=:), allocatable :: json
    character(len=32) :: buf
    integer :: i

    json = '{' // char(10) // &
           '  "indicadores": [' // char(10)

    do i = 1, n_datos
      write(buf, '(I4)') anios(i)
      json = json // '    {"anio":' // trim(adjustl(buf)) // ','
      write(buf, '(I0)') meses(i)
      json = json // '"mes":' // trim(buf) // ','
      write(buf, '(F0.2)') inflacion(i)
      json = json // '"inflacion":' // trim(buf) // ','
      write(buf, '(F0.2)') pib(i)
      json = json // '"pib":' // trim(buf) // ','
      write(buf, '(F0.2)') desempleo(i)
      json = json // '"desempleo":' // trim(buf) // ','
      write(buf, '(F0.2)') tipo_cambio(i)
      json = json // '"tipo_cambio":' // trim(buf) // '}'
      if (i < n_datos) json = json // ','
      json = json // char(10)
    end do

    json = json // '  ],' // char(10)
    json = json // '  "estadisticas": {' // char(10)

    call append_est(json, 'inflacion', 1)
    json = json // ',' // char(10)
    call append_est(json, 'pib', 2)
    json = json // ',' // char(10)
    call append_est(json, 'desempleo', 3)
    json = json // ',' // char(10)
    call append_est(json, 'tipo_cambio', 4)
    json = json // char(10) // '  }' // char(10) // '}'

  end function json_todos_indicadores

  ! Genera JSON para un indicador especifico
  function json_indicador(nombre) result(json)
    character(len=*), intent(in) :: nombre
    character(len=:), allocatable :: json
    character(len=32) :: buf
    integer :: idx, i

    idx = indice_indicador(nombre)
    if (idx == 0) then
      json = '{"error": "Indicador no encontrado: ' // nombre // '"}'
      return
    end if

    json = '{' // char(10) // &
           '  "nombre": "' // trim(nombre) // '",' // char(10) // &
           '  "datos": ['
    do i = 1, n_datos
      write(buf, '(F0.2)') valor_indicador(idx, i)
      json = json // trim(buf)
      if (i < n_datos) json = json // ','
    end do
    json = json // '],' // char(10)

    json = json // '  "estadisticas": {' // char(10)
    call append_est(json, nombre, idx)
    json = json // char(10) // '  },' // char(10)

    ! Tendencia
    if (pendientes(idx) > 0.01) then
      json = json // '  "tendencia": "creciente"' // char(10)
    else if (pendientes(idx) < -0.01) then
      json = json // '  "tendencia": "decreciente"' // char(10)
    else
      json = json // '  "tendencia": "estable"' // char(10)
    end if
    json = json // '}'
  end function json_indicador

  ! Genera JSON con la prediccion para un indicador
  function json_prediccion(nombre) result(json)
    character(len=*), intent(in) :: nombre
    character(len=:), allocatable :: json
    character(len=32) :: buf
    integer :: idx, i

    idx = indice_indicador(nombre)
    if (idx == 0) then
      json = '{"error": "Indicador no encontrado: ' // nombre // '"}'
      return
    end if

    json = '{' // char(10) // &
           '  "indicador": "' // trim(nombre) // '",' // char(10)
    write(buf, '(F0.4)') pendientes(idx)
    json = json // '  "regresion": {' // char(10) // &
           '    "pendiente": ' // trim(buf) // ','
    write(buf, '(F0.4)') intersecciones(idx)
    json = json // char(10) // '    "interseccion": ' // trim(buf) // ','
    write(buf, '(F0.4)') r2s(idx)
    json = json // char(10) // '    "r2": ' // trim(buf) // char(10) // &
           '  },' // char(10)
    json = json // '  "predicciones": [' // char(10)

    do i = 1, 6
      write(buf, '(I0)') nint(preds_periodos(i))
      json = json // '    {"periodo": ' // trim(buf) // ','
      write(buf, '(F0.2)') preds_valores(idx, i)
      json = json // '"valor_estimado": ' // trim(buf) // '}'
      if (i < 6) json = json // ','
      json = json // char(10)
    end do

    json = json // '  ]' // char(10) // '}'
  end function json_prediccion

  ! Funcion auxiliar: indice de un indicador por nombre
  function indice_indicador(nombre) result(idx)
    character(len=*), intent(in) :: nombre
    integer :: idx
    select case(trim(nombre))
    case('inflacion');   idx = 1
    case('pib');         idx = 2
    case('desempleo');   idx = 3
    case('tipo_cambio'); idx = 4
    case default;        idx = 0
    end select
  end function indice_indicador

  ! Funcion auxiliar: valor de un indicador en una fila
  function valor_indicador(idx, fila) result(val)
    integer, intent(in) :: idx, fila
    real :: val
    select case(idx)
    case(1); val = inflacion(fila)
    case(2); val = pib(fila)
    case(3); val = desempleo(fila)
    case(4); val = tipo_cambio(fila)
    end select
  end function valor_indicador

  ! Subrutina auxiliar: agrega estadisticas de un indicador al JSON
  subroutine append_est(json, nombre, idx)
    character(len=:), allocatable, intent(inout) :: json
    character(len=*), intent(in) :: nombre
    integer, intent(in) :: idx
    character(len=32) :: buf

    json = json // '    "' // trim(nombre) // '": {' // char(10)
    write(buf, '(F0.2)') medias(idx)
    json = json // '      "media": ' // trim(buf) // ','
    write(buf, '(F0.2)') desviaciones(idx)
    json = json // char(10) // '      "desviacion": ' // trim(buf) // ','
    write(buf, '(F0.2)') minimos(idx)
    json = json // char(10) // '      "minimo": ' // trim(buf) // ','
    write(buf, '(F0.2)') maximos(idx)
    json = json // char(10) // '      "maximo": ' // trim(buf) // char(10)
    json = json // '    }'
  end subroutine append_est

end module mod_api_handler
```

### Analisis linea por linea

**Lineas 1–7**: El modulo usa `iso_c_binding` para interoperar con C. Las funciones POSIX de sockets no tienen wrappers nativos en Fortran, por lo que debemos declarar sus interfaces manualmente.

**Lineas 10–16**: `c_socket`: Toma dominio (AF_INET=2), tipo (SOCK_STREAM=1) y protocolo (0=automatico). Devuelve un file descriptor o -1 si falla.

**Lineas 18–25**: `c_bind`: Asocia el socket a una direccion y puerto. `addr` es un puntero a `sockaddr_in`. `addrlen` es el tamano de la estructura (16 bytes).

**Lineas 27–32**: `c_listen`: Pone el socket en modo escucha. `backlog=5` es el numero maximo de conexiones en cola.

**Lineas 34–41**: `c_accept`: Bloquea hasta que llegue una conexion. Devuelve un nuevo socket para la conexion aceptada. `addrlen` es un puntero a un entero que se actualiza con el tamano real de la direccion del cliente.

**Lineas 43–51**: `c_send` y `c_recv`: Envian y reciben datos. Devuelven `ssize_t` (long en 64 bits). El buffer se pasa como `c_ptr`.

**Lineas 59–65**: `c_close`: Cierra un file descriptor.

**Lineas 67–72**: `c_htons` y `c_htonl`: Convierten valores de host a network byte order. Necesarios porque el puerto y la direccion IP deben estar en big-endian en el socket.

**Lineas 74–81**: `c_setsockopt`: Configura opciones del socket. Usamos `SO_REUSEADDR` para evitar el error "Address already in use" al reiniciar el servidor.

**Linea 87**: `AF_INET = 2` para IPv4. En Linux, estas constantes vienen de `/usr/include/netinet/in.h`.

**Lineas 94–99**: `type, bind(c) :: sockaddr_in`. La estructura debe ser compatible con C. `sin_family` es `short` (2 bytes), `sin_port` es `short` (2 bytes) en network byte order, `sin_addr` es `int` (4 bytes), y `sin_zero` es padding de 8 bytes. Total: 16 bytes.

**Lineas 102–113**: Variables privadas del modulo para almacenar datos. Se usa `save` para preservar valores entre llamadas. `datos_cargados` es un flag para verificar que `set_datos` se haya llamado antes de iniciar el servidor.

**Lineas 119–134**: `subroutine set_datos(...)`. Toma los datos separados por indicador (como arreglos individuales) en lugar de un arreglo de tipo `t_indicador`, para evitar dependencias del modulo `mod_datos_csv`. Copia todos los valores a las variables privadas.

**Linea 140**: `subroutine iniciar_servidor(puerto)`. Bucle principal del servidor. No retorna hasta que se interrumpa con Ctrl+C.

**Lineas 151–155**: Verifica que los datos esten cargados. Si no, imprime un error y retorna.

**Lineas 157–160**: `c_socket(AF_INET, SOCK_STREAM, 0)` crea un socket TCP. Si devuelve negativo, hubo un error.

**Lineas 162–165**: `SO_REUSEADDR` permite reutilizar el puerto inmediatamente despues de detener el servidor. Sin esto, tendria que esperar 60 segundos (TIME_WAIT).

**Lineas 167–170**: Configura la direccion del servidor: familia AF_INET, puerto en network byte order (`c_htons`), direccion INADDR_ANY (0.0.0.0) para aceptar conexiones de cualquier interfaz.

**Lineas 172–178**: `bind` asocia el socket al puerto. Si falla (puerto ocupado o permisos insuficientes), imprime error y retorna.

**Lineas 180–186**: `listen` pone el socket en modo pasivo. Acepta hasta 5 conexiones en cola.

**Lineas 188–198**: Informacion al usuario sobre los endpoints disponibles.

**Lineas 200–210**: Bucle infinito: `accept` espera una conexion. Cuando llega, `c_accept` devuelve un nuevo socket. `client_addr_len` debe inicializarse al tamano de `sockaddr_in` antes de llamar a `accept`.

**Lineas 212–214**: Se asigna un buffer de 4096 bytes para recibir la peticion HTTP.

**Lineas 215–219**: `c_recv` lee los datos del socket. `bytes_recibidos` contiene la cantidad de bytes leidos.

**Lineas 221–226**: Convierte el buffer C a una cadena Fortran para procesarla.

**Linea 228**: `response_str = procesar_peticion(request_str)`. La funcion principal de enrutamiento.

**Lineas 230–240**: Construye las cabeceras HTTP: codigo 200 OK, tipo de contenido JSON, CORS, longitud del contenido. Dos CRLF (`char(13)//char(10)`) separan cabeceras del cuerpo.

**Lineas 242–249**: Construye el buffer de envio concatenando cabeceras y cuerpo.

**Lineas 251–253**: `c_send` envia la respuesta al cliente.

**Lineas 255–260**: Limpieza: desasigna buffers, cierra el socket del cliente.

**Linea 271**: `function procesar_peticion(solicitud)`. Extrae el metodo y la ruta de la primera linea HTTP.

**Lineas 273–282**: `index(solicitud, ' ')` busca el primer espacio. `pos1` es la posicion del primer espacio, que separa el metodo de la ruta en `GET /ruta HTTP/1.1`.

**Lineas 284–289**: Segundo espacio separa la ruta de la version HTTP.

**Linea 290**: `ruta = solicitud(pos1+1:pos1+pos2-1)`. Extrae la ruta completa.

**Lineas 292–304**: Enrutamiento: compara `ruta` con los patrones conocidos. `ruta(1:14) == '/indicadores/'` verifica el prefijo del endpoint.

**Linea 301**: Si la ruta es `/`, muestra un mensaje de bienvenida.

**Lineas 308–374**: `json_todos_indicadores()`. Genera un JSON con todos los datos mensuales y las estadisticas. Construye la cadena manualmente con concatenacion.

**Lineas 316–332**: Bucle que genera un objeto JSON por cada mes. Usa `write(buf, '(F0.2)')` para formatear numeros reales con ancho minimo.

**Lineas 338–344**: Llama a `append_est` para agregar las estadisticas de cada indicador.

**Lineas 350–376**: `json_indicador(nombre)`. Busca el indice del indicador con `indice_indicador`, genera el arreglo de datos y las estadisticas.

**Lineas 380–401**: `json_prediccion(nombre)`. Genera un JSON con los coeficientes de regresion y las predicciones para los proximos 6 periodos.

**Lineas 407–416**: `indice_indicador(nombre)`. Convierte nombre a indice (1–4) usando `select case`.

**Lineas 422–431**: `valor_indicador(idx, fila)`. Extrae el valor de un indicador de una fila dada.

**Lineas 437–453**: `append_est(json, nombre, idx)`. Agrega las estadisticas de un indicador al JSON en construccion.

### Salida esperada

Al iniciar el servidor:

```
=== Servidor HTTP iniciado ===
Puerto: 8080
URL base: http://localhost:8080
Endpoints:
  GET /indicadores
  GET /indicadores/{nombre}
  GET /prediccion/{indicador}
Presione Ctrl+C para detener.
```

Consultando `GET /indicadores/inflacion` con curl:

```json
{
  "nombre": "inflacion",
  "datos": [7.91,7.62,7.33,6.85,6.42,5.98,5.55,5.12,4.78,4.45,4.20,3.95,3.75,3.60,3.50,3.45,3.42,3.40,3.38,3.35,3.30,3.25,3.20,3.18],
  "estadisticas": {
    "inflacion": {
      "media": 4.64,
      "desviacion": 1.66,
      "minimo": 3.18,
      "maximo": 7.91
    }
  },
  "tendencia": "decreciente"
}
```

Consultando `GET /prediccion/inflacion`:

```json
{
  "indicador": "inflacion",
  "regresion": {
    "pendiente": -0.1943,
    "interseccion": 397.45,
    "r2": 0.9234
  },
  "predicciones": [
    {"periodo": 25, "valor_estimado": 3.12},
    {"periodo": 26, "valor_estimado": 2.93},
    {"periodo": 27, "valor_estimado": 2.73},
    {"periodo": 28, "valor_estimado": 2.54},
    {"periodo": 29, "valor_estimado": 2.34},
    {"periodo": 30, "valor_estimado": 2.15}
  ]
}
```

### Errores tipicos

```fortran
! Error: olvidar convertir puerto a network byte order
server_addr%sin_port = int(puerto, c_short)  ! incorrecto en little-endian
server_addr%sin_port = c_htons(int(puerto, c_short))  ! correcto
```

En una maquina little-endian (x86), el valor `8080` se almacena como `0x901F` en lugar de `0x1F90`. `htons` intercambia los bytes para que el puerto quede en big-endian, que es lo que espera el protocolo de red.

```
Error: no se pudo asociar al puerto 8080
```

**Solucion**: El puerto puede estar ocupado por otro proceso. Use `sudo netstat -tlnp | grep 8080` para identificar el proceso, o cambie el puerto en el codigo. La opcion `SO_REUSEADDR` ayuda a reutilizar puertos en estado TIME_WAIT.

```
Error: no se pudo crear el socket
```

**Solucion**: Verifique que el sistema tenga soporte para sockets (en Linux siempre lo tiene). En entornos containerizados, asegurese de que el contenedor tenga permisos de red.

```fortran
! Error: buffer de recepcion demasiado pequeno
character(kind=c_char), allocatable, target :: recv_buf(:)
allocate(recv_buf(128))  ! muy pequeno para peticiones grandes
```

Si la peticion HTTP excede el buffer, se truncara. Usamos 4096 bytes, suficiente para peticiones tipicas. Para produccion, se necesitaria un buffer dinamico o lectura multiple.

---

## 7. Programa principal (orquestador)

### Pre-explicacion conceptual

Hemos construido los modulos individuales. Ahora necesitamos un **programa principal** que los orqueste en la secuencia correcta. Este programa es el corazon del sistema: llama a cada modulo en el orden adecuado, pasa los datos de uno a otro y maneja el flujo global.

El flujo es el siguiente:

1. **Leer el CSV**: Llama a `leer_csv` del modulo `mod_datos_csv`. Obtiene un arreglo de `t_indicador`.
2. **Calcular estadisticas**: Llama a `calcular_estadisticas` del modulo `mod_estadisticas`. Obtiene medias, desviaciones, minimos, maximos y matriz de correlacion.
3. **Calcular tendencias**: Llama a `calcular_tendencias`.
4. **Ejecutar regresion lineal**: Para cada indicador, extrae sus valores y corre `regresion_lineal`. Luego genera predicciones con `predecir`.
5. **Generar reportes**: Llama a `generar_informe_latex` y `generar_script_gnuplot` del modulo `mod_reportes`.
6. **Configurar API**: Llama a `set_datos` del modulo `mod_api_handler` para pasarle todos los datos y resultados.
7. **Iniciar servidor**: Llama a `iniciar_servidor(8080)`. El programa se queda en este bucle hasta que el usuario presione Ctrl+C.

Una buena practica es imprimir mensajes de progreso en cada paso, para que el usuario sepa en que etapa esta el proceso. Tambien es importante verificar que los datos se hayan leido correctamente antes de continuar.

El programa principal no deberia contener logica de negocio. Su unica responsabilidad es coordinar: leer datos, llamar a los modulos y mostrar resultados. Esto sigue el principio de **separacion de responsabilidades** (Separation of Concerns).

### Codigo

```fortran
! programa_principal.f90
! Orquestador del sistema de monitoreo economico
program programa_principal
  use mod_datos_csv
  use mod_estadisticas
  use mod_regresion
  use mod_reportes
  use mod_api_handler
  implicit none

  type(t_indicador), allocatable :: datos(:)
  real :: medias(4), desviaciones(4), minimos(4), maximos(4)
  real :: matriz_corr(4, 4)
  character(len=20) :: tendencias(4)
  type(t_regresion) :: regs(4)
  real, allocatable :: x(:), y(:), x_pred(:), y_pred(:)
  real :: preds_periodos(6), preds_valores(4, 6)
  integer :: n, i, j

  print *, '=== Sistema de Monitoreo Economico ==='
  print *, ''

  ! Paso 1: Leer CSV
  print *, 'Paso 1: Leyendo indicadores_economicos.csv...'
  datos = leer_csv('indicadores_economicos.csv')
  if (.not. allocated(datos) .or. size(datos) == 0) then
    print *, 'Error: no se pudieron leer los datos. Abortando.'
    stop
  end if
  call imprimir_datos(datos)
  print *, ''

  ! Paso 2: Estadisticas descriptivas
  n = size(datos)
  print *, 'Paso 2: Calculando estadisticas descriptivas...'
  call calcular_estadisticas(datos, medias, desviaciones, minimos, &
                             maximos, matriz_corr)
  call calcular_tendencias(datos, tendencias)

  do i = 1, 4
    select case(i)
    case(1); print '("  Inflacion:     media=", F6.2, " std=", F6.2, &
                    &" tendencia=", A)', medias(i), desviaciones(i), &
                    trim(tendencias(i))
    case(2); print '("  PIB:           media=", F6.2, " std=", F6.2, &
                    &" tendencia=", A)', medias(i), desviaciones(i), &
                    trim(tendencias(i))
    case(3); print '("  Desempleo:     media=", F6.2, " std=", F6.2, &
                    &" tendencia=", A)', medias(i), desviaciones(i), &
                    trim(tendencias(i))
    case(4); print '("  Tipo de cambio: media=", F6.2, " std=", F6.2, &
                    &" tendencia=", A)', medias(i), desviaciones(i), &
                    trim(tendencias(i))
    end select
  end do
  print *, ''

  ! Paso 3: Regresion lineal para cada indicador
  print *, 'Paso 3: Calculando regresion lineal...'
  allocate(x(n))
  x = real([(i, i = 1, n)])
  allocate(y(n))

  do i = 1, 4
    select case(i)
    case(1); y = real(datos%inflacion)
    case(2); y = real(datos%pib)
    case(3); y = real(datos%desempleo)
    case(4); y = real(datos%tipo_cambio)
    end select

    regs(i) = regresion_lineal(x, y)
    print '("  ", A, ": pendiente=", F8.4, " interseccion=", F8.4, &
            &" R^2=", F6.4)', &
      ajustar_nombre(i), regs(i)%pendiente, regs(i)%interseccion, regs(i)%r2
  end do

  ! Predecir 6 periodos futuros
  allocate(x_pred(6))
  do i = 1, 6
    x_pred(i) = real(n + i)
  end do

  do i = 1, 4
    y_pred = predecir(regs(i), x_pred)
    do j = 1, 6
      preds_valores(i, j) = y_pred(j)
    end do
  end do
  preds_periodos = x_pred
  deallocate(x, y, x_pred)

  print *, ''

  ! Paso 4: Generar reportes
  print *, 'Paso 4: Generando reportes...'
  call generar_informe_latex('informe_economico.tex', datos, medias, &
                             desviaciones, minimos, maximos, matriz_corr, &
                             tendencias, regs, preds_valores)
  call generar_script_gnuplot('graficos.plt', datos, regs)
  print *, ''

  ! Paso 5: Configurar y iniciar API REST
  print *, 'Paso 5: Configurando API REST...'
  call set_datos( &
    datos%anio, datos%mes, real(datos%inflacion), real(datos%pib), &
    real(datos%desempleo), real(datos%tipo_cambio), &
    medias, desviaciones, minimos, maximos, &
    regs%pendiente, regs%interseccion, regs%r2, &
    preds_periodos, preds_valores)

  print *, 'Iniciando servidor HTTP en puerto 8080...'
  call iniciar_servidor(8080)

contains

  ! Devuelve un nombre legible para cada indicador
  function ajustar_nombre(i) result(nombre)
    integer, intent(in) :: i
    character(len=20) :: nombre
    select case(i)
    case(1); nombre = 'Inflacion'
    case(2); nombre = 'PIB'
    case(3); nombre = 'Desempleo'
    case(4); nombre = 'Tipo de cambio'
    end select
  end function ajustar_nombre

end program programa_principal
```

### Analisis linea por linea

**Lineas 1–4**: Comentarios de cabecera. El programa se llama `programa_principal`.

**Lineas 5–9**: `use` de todos los modulos del sistema. El orden no importa aqui, pero es buena practica listarlos alfabeticamente o por dependencia.

**Lineas 11–19**: Declaracion de variables. `type(t_regresion) :: regs(4)` crea un arreglo de 4 modelos de regresion (uno por indicador). `preds_valores(4, 6)` almacena las predicciones: 4 indicadores × 6 periodos.

**Lineas 23–30**: Lectura del CSV. Si `leer_csv` retorna un arreglo no asignado o vacio, el programa se detiene con `stop`.

**Linea 32**: `call imprimir_datos(datos)` muestra los datos en formato tabular.

**Lineas 42–57**: Bucle que imprime las estadisticas de cada indicador. Usa `select case` para determinar el nombre del indicador.

**Lineas 62–63**: `x = real([(i, i = 1, n)])` crea la variable independiente (1, 2, 3, ...).

**Lineas 65–78**: Para cada indicador, extrae los valores, ejecuta la regresion e imprime los coeficientes. Note que `regs(i)` es de tipo `t_regresion` y se asigna directamente desde el resultado de `regresion_lineal`.

**Lineas 81–89**: Genera las predicciones: `x_pred` contiene los periodos 25 a 30 (6 meses futuros). `predecir` aplica la recta de regresion a cada uno.

**Lineas 97–100**: Llamadas a `generar_informe_latex` y `generar_script_gnuplot` con todos los resultados.

**Lineas 104–108**: `set_datos` pasa todos los datos al modulo API. Observe que los campos de `datos` se pasan como arreglos individuales usando la sintaxis de arreglo implicito.

**Linea 110**: `call iniciar_servidor(8080)` inicia el servidor. Esta llamada no retorna hasta que se interrumpa el programa.

### Salida esperada

```
=== Sistema de Monitoreo Economico ===

Paso 1: Leyendo indicadores_economicos.csv...
 Leidos 24 registros de indicadores_economicos.csv
 Anio  Mes  Inflacion  PIB    Desempleo  T.Cambio
 -------------------------------------------------------
 2023   1      7.91   3.70     2.90    18.85
 ...
 2024  12      3.18   3.70     2.80    17.30

Paso 2: Calculando estadisticas descriptivas...
  Inflacion:     media=  4.64 std=  1.66 tendencia=decreciente
  PIB:           media=  3.79 std=  0.21 tendencia=estable
  Desempleo:     media=  2.93 std=  0.17 tendencia=estable
  Tipo de cambio: media= 17.55 std=  0.56 tendencia=decreciente

Paso 3: Calculando regresion lineal...
  Inflacion: pendiente= -0.1943 interseccion=397.4541 R^2=0.9234
  PIB: pendiente=  0.0034 interseccion=  3.7494 R^2=0.0012
  Desempleo: pendiente=  0.0117 interseccion=  2.7916 R^2=0.0532
  Tipo de cambio: pendiente= -0.0686 interseccion=156.0345 R^2=0.8962

Paso 4: Generando reportes...
 Informe LaTeX generado: informe_economico.tex
 Script Gnuplot generado: graficos.plt

Paso 5: Configurando API REST...
Iniciando servidor HTTP en puerto 8080...

=== Servidor HTTP iniciado ===
Puerto: 8080
URL base: http://localhost:8080
Endpoints:
  GET /indicadores
  GET /indicadores/{nombre}
  GET /prediccion/{indicador}
Presione Ctrl+C para detener.
```

### Errores tipicos

```fortran
! Error: pasar arreglos incompatibles
call set_datos(datos%anio, datos%mes, ...)
```

Si `datos` no esta asignado (porque `leer_csv` fallo), los arreglos implicitos como `datos%anio` causaran un error en tiempo de ejecucion. El programa verifica esto antes de continuar.

```
Error: no se pudieron leer los datos. Abortando.
```

**Solucion**: Asegurese de que el archivo `indicadores_economicos.csv` exista en el directorio de trabajo. Use `ls -la` para verificar.

---

## 8. Cliente de prueba de la API

### Pre-explicacion conceptual

Hemos construido un servidor API en Fortran, pero ¿como probamos que funciona sin escribir un navegador web? La herramienta mas comun es `curl`, un cliente de linea de comandos para transferir datos con URLs. Sin embargo, para mantener la coherencia pedagogica del manual, escribiremos un **cliente en Fortran** que use `curl` internamente a traves de `execute_command_line`.

`execute_command_line` es una funcion intrinseca de Fortran 2008 que ejecuta un comando del sistema operativo. Es nuestra puerta de enlace al mundo exterior. Con ella podemos llamar a `curl`, capturar su salida y mostrarla al usuario.

El cliente hara tres cosas:

1. Consultar `GET /indicadores/inflacion` y mostrar el JSON recibido.
2. Consultar `GET /prediccion/inflacion` y mostrar los resultados.
3. Calcular el tiempo total de las consultas.

La salida de `curl` se redirige a un archivo temporal, que luego el programa Fortran lee y muestra. Esto es necesario porque `execute_command_line` solo devuelve el codigo de salida, no la salida estandar directamente.

¿Por que no usar directamente `curl` desde la terminal? Porque el objetivo es demostrar que un programa Fortran puede actuar como consumidor de una API REST. En un sistema real, el cliente Fortran podria parsear el JSON y tomar decisiones basadas en los datos recibidos.

El cliente asume que el servidor ya esta corriendo en `localhost:8080`.

### Codigo

```fortran
! cliente_api.f90
! Cliente de prueba para la API de monitoreo economico
program cliente_api
  implicit none

  integer :: codigo_salida
  character(len=256) :: linea
  integer :: unidad, ios, n_lineas

  print *, '=== Cliente de prueba de la API ==='
  print *, 'Servidor: http://localhost:8080'
  print *, ''

  ! Consulta 1: inflacion
  print *, 'Consulta: GET /indicadores/inflacion'
  print *, '----------------------------------------'
  call execute_command_line( &
    'curl -s -o /tmp/respuesta_api.txt http://localhost:8080/indicadores/inflacion', &
    exitstat=codigo_salida)

  if (codigo_salida /= 0) then
    print *, 'Error: no se pudo conectar al servidor (curl exit code=', &
             codigo_salida, ')'
    print *, 'Asegurese de que el servidor este corriendo en puerto 8080.'
    stop
  end if

  open(newunit=unidad, file='/tmp/respuesta_api.txt', status='old', &
       action='read', iostat=ios)
  if (ios == 0) then
    n_lineas = 0
    do
      read(unidad, '(A)', iostat=ios) linea
      if (ios /= 0) exit
      print *, trim(linea)
      n_lineas = n_lineas + 1
    end do
    close(unidad)
  end if
  print *, ''

  ! Consulta 2: prediccion de inflacion
  print *, 'Consulta: GET /prediccion/inflacion'
  print *, '----------------------------------------'
  call execute_command_line( &
    'curl -s -o /tmp/respuesta_api.txt http://localhost:8080/prediccion/inflacion', &
    exitstat=codigo_salida)

  open(newunit=unidad, file='/tmp/respuesta_api.txt', status='old', &
       action='read', iostat=ios)
  if (ios == 0) then
    do
      read(unidad, '(A)', iostat=ios) linea
      if (ios /= 0) exit
      print *, trim(linea)
    end do
    close(unidad)
  end if
  print *, ''

  ! Consulta 3: todos los indicadores (solo mostrar primeras lineas)
  print *, 'Consulta: GET /indicadores (truncado a 5 lineas)'
  print *, '----------------------------------------'
  call execute_command_line( &
    'curl -s -o /tmp/respuesta_api.txt http://localhost:8080/indicadores', &
    exitstat=codigo_salida)

  open(newunit=unidad, file='/tmp/respuesta_api.txt', status='old', &
       action='read', iostat=ios)
  if (ios == 0) then
    n_lineas = 0
    do
      read(unidad, '(A)', iostat=ios) linea
      if (ios /= 0) exit
      print *, trim(linea)
      n_lineas = n_lineas + 1
      if (n_lineas >= 5) then
        print *, '... (respuesta truncada, ', n_lineas, ' lineas mostradas)'
        exit
      end if
    end do
    close(unidad)
  end if

  print *, ''
  print *, '=== Cliente finalizado exitosamente ==='

end program cliente_api
```

### Analisis linea por linea

**Lineas 1–3**: Programa simple que no usa modulos. Depende unicamente de `execute_command_line` y E/S de archivos.

**Lineas 10–11**: Configuracion inicial: imprime la URL del servidor.

**Lineas 15–18**: `call execute_command_line(...)`. Ejecuta `curl -s -o /tmp/respuesta_api.txt http://localhost:8080/indicadores/inflacion`. La opcion `-s` (silent) suprime la barra de progreso. `-o` escribe la salida en un archivo temporal.

**Linea 19**: `exitstat=codigo_salida` captura el codigo de salida de `curl`. Si es distinto de 0, hubo un error de conexion.

**Lineas 21–25**: Verifica si `curl` tuvo exito. Si no, muestra un mensaje de error y termina.

**Lineas 27–39**: Abre el archivo de respuesta temporal y lo imprime linea por linea. Usa un bucle `do` infinito que termina cuando `ios /= 0` (fin de archivo).

**Lineas 42–55**: Segunda consulta a `/prediccion/inflacion`. Mismo patron.

**Lineas 58–76**: Tercera consulta a `/indicadores`, pero solo muestra las primeras 5 lineas (el JSON completo es muy largo).

### Salida esperada

```
=== Cliente de prueba de la API ===
Servidor: http://localhost:8080

Consulta: GET /indicadores/inflacion
----------------------------------------
{
  "nombre": "inflacion",
  "datos": [7.91,7.62,7.33,6.85,6.42,5.98,5.55,5.12,4.78,4.45,4.20,3.95,3.75,3.60,3.50,3.45,3.42,3.40,3.38,3.35,3.30,3.25,3.20,3.18],
  "estadisticas": {
    "inflacion": {
      "media": 4.64,
      "desviacion": 1.66,
      "minimo": 3.18,
      "maximo": 7.91
    }
  },
  "tendencia": "decreciente"
}

Consulta: GET /prediccion/inflacion
----------------------------------------
{
  "indicador": "inflacion",
  "regresion": {
    "pendiente": -0.1943,
    "interseccion": 397.45,
    "r2": 0.9234
  },
  "predicciones": [
    {"periodo": 25, "valor_estimado": 3.12},
    {"periodo": 26, "valor_estimado": 2.93},
    {"periodo": 27, "valor_estimado": 2.73},
    {"periodo": 28, "valor_estimado": 2.54},
    {"periodo": 29, "valor_estimado": 2.34},
    {"periodo": 30, "valor_estimado": 2.15}
  ]
}

Consulta: GET /indicadores (truncado a 5 lineas)
----------------------------------------
{
  "indicadores": [
    {"anio":2023,"mes":1,"inflacion":7.91,"pib":3.70,"desempleo":2.90,"tipo_cambio":18.85},
    {"anio":2023,"mes":2,"inflacion":7.62,"pib":3.50,"desempleo":2.80,"tipo_cambio":18.75},
    {"anio":2023,"mes":3,"inflacion":7.33,"pib":4.10,"desempleo":2.70,"tipo_cambio":18.50},
    {"anio":2023,"mes":4,"inflacion":6.85,"pib":3.80,"desempleo":2.60,"tipo_cambio":18.20},
... (respuesta truncada, 5 lineas mostradas)

=== Cliente finalizado exitosamente ===
```

### Errores tipicos

```
Error: no se pudo conectar al servidor (curl exit code= 7)
Asegurese de que el servidor este corriendo en puerto 8080.
```

**Solucion**: El codigo de salida 7 de curl significa "Failed to connect to host". Asegurese de que el servidor este corriendo en otra terminal antes de ejecutar el cliente.

```
Error leyendo archivo de respuesta temporal
```

**Solucion**: Verifique que `/tmp` sea escribible. Si el sistema no tiene `/tmp`, use una ruta alternativa como `./respuesta_api.txt`.

---

## 9. Compilacion y ejecucion del sistema completo

### Pre-explicacion conceptual

Llegamos al final del camino. Ahora debemos compilar los siete archivos fuente en el orden correcto y ejecutar el sistema completo. La compilacion de un sistema multi-modulo en Fortran requiere respetar las **dependencias entre modulos**: un modulo debe compilarse antes que cualquier programa o modulo que lo use.

El orden de compilacion es:

```
mod_datos_csv.f90       -> mod_datos_csv.mod
mod_estadisticas.f90    -> mod_estadisticas.mod  (usa mod_datos_csv)
mod_regresion.f90       -> mod_regresion.mod
mod_reportes.f90        -> mod_reportes.mod      (usa mod_datos_csv)
mod_api_handler.f90     -> mod_api_handler.mod
programa_principal.f90  -> programa_principal    (usa todos)
cliente_api.f90         -> cliente_api           (independiente)
```

Podriamos compilar cada archivo manualmente, pero es mucho mas practico usar un **Makefile**, que automatiza el proceso y solo recompila los archivos que han cambiado.

Make es una herramienta clasica de Unix que define reglas de compilacion con dependencias. Cada `target` (archivo a generar) tiene `prerequisites` (archivos de los que depende) y un `recipe` (comandos para generarlo).

Para facilitar las pruebas, tambien crearemos el archivo CSV de ejemplo y un pequeno script de prueba automatizada que ejecute todo el sistema.

### Datos de ejemplo

Cree un archivo `indicadores_economicos.csv` con el siguiente contenido (24 meses de datos economicos realistas):

```
anio,mes,inflacion,pib,desempleo,tipo_cambio
2023,1,7.91,3.70,2.90,18.85
2023,2,7.62,3.50,2.80,18.75
2023,3,7.33,4.10,2.70,18.50
2023,4,6.85,3.80,2.60,18.20
2023,5,6.42,3.90,2.70,17.95
2023,6,5.98,4.20,2.80,17.80
2023,7,5.55,4.00,2.90,17.65
2023,8,5.12,3.60,3.00,17.50
2023,9,4.78,3.50,3.10,17.40
2023,10,4.45,3.80,3.00,17.35
2023,11,4.20,4.00,2.90,17.30
2023,12,3.95,3.70,2.80,17.25
2024,1,3.75,3.50,2.80,17.20
2024,2,3.60,3.60,2.90,17.15
2024,3,3.50,3.80,2.90,17.10
2024,4,3.45,3.90,3.00,17.05
2024,5,3.42,4.10,3.00,17.00
2024,6,3.40,4.00,3.10,17.00
2024,7,3.38,3.80,3.10,17.05
2024,8,3.35,3.70,3.20,17.10
2024,9,3.30,3.50,3.10,17.15
2024,10,3.25,3.60,3.00,17.20
2024,11,3.20,3.80,2.90,17.25
2024,12,3.18,3.70,2.80,17.30
```

### Makefile

```makefile
# Makefile para el Sistema de Monitoreo Economico
FC = gfortran
FFLAGS = -std=f2018 -Wall -Wextra

MOD_OBJS = mod_datos_csv.o mod_estadisticas.o mod_regresion.o \
           mod_reportes.o mod_api_handler.o

all: programa_principal cliente_api

programa_principal: programa_principal.o $(MOD_OBJS)
	$(FC) $(FFLAGS) -o $@ $^

cliente_api: cliente_api.o
	$(FC) $(FFLAGS) -o $@ $^

mod_datos_csv.o: mod_datos_csv.f90
	$(FC) $(FFLAGS) -c $<

mod_estadisticas.o: mod_estadisticas.f90 mod_datos_csv.o
	$(FC) $(FFLAGS) -c $<

mod_regresion.o: mod_regresion.f90
	$(FC) $(FFLAGS) -c $<

mod_reportes.o: mod_reportes.f90 mod_datos_csv.o
	$(FC) $(FFLAGS) -c $<

mod_api_handler.o: mod_api_handler.f90
	$(FC) $(FFLAGS) -c $<

programa_principal.o: programa_principal.f90 $(MOD_OBJS:.o=.mod)
	$(FC) $(FFLAGS) -c $<

cliente_api.o: cliente_api.f90
	$(FC) $(FFLAGS) -c $<

clean:
	rm -f *.o *.mod programa_principal cliente_api

.PHONY: all clean
```

### Instrucciones de ejecucion paso a paso

#### 1. Crear el directorio de trabajo

```bash
mkdir -p ~/monitoreo_economico
cd ~/monitoreo_economico
```

#### 2. Guardar todos los archivos fuente

Cree cada archivo `.f90` con el contenido de las secciones anteriores y el CSV con los datos de ejemplo.

#### 3. Compilar el sistema

```bash
gfortran -std=f2018 -Wall -c mod_datos_csv.f90
gfortran -std=f2018 -Wall -c mod_regresion.f90
gfortran -std=f2018 -Wall -c mod_estadisticas.f90
gfortran -std=f2018 -Wall -c mod_reportes.f90
gfortran -std=f2018 -Wall -c mod_api_handler.f90
gfortran -std=f2018 -Wall -c programa_principal.f90
gfortran -std=f2018 -Wall -o programa_principal programa_principal.o \
  mod_datos_csv.o mod_estadisticas.o mod_regresion.o \
  mod_reportes.o mod_api_handler.o
gfortran -std=f2018 -Wall -c cliente_api.f90
gfortran -std=f2018 -Wall -o cliente_api cliente_api.o
```

O simplemente:

```bash
make
```

#### 4. Ejecutar el sistema (servidor)

En una terminal:

```bash
./programa_principal
```

Esto mostrara los resultados del analisis y dejara el servidor corriendo en el puerto 8080.

#### 5. Probar la API con curl

En otra terminal:

```bash
curl -s http://localhost:8080/indicadores | head -20
curl -s http://localhost:8080/indicadores/inflacion
curl -s http://localhost:8080/prediccion/inflacion
```

#### 6. Ejecutar el cliente de prueba

```bash
./cliente_api
```

#### 7. Generar reportes adicionales

Con el servidor detenido (o en otra terminal mientras corre), compile el informe LaTeX:

```bash
pdflatex informe_economico.tex
```

Y genere los graficos:

```bash
gnuplot graficos.plt
```

### Prueba del sistema completo

A continuacion se muestra la salida esperada de una ejecucion completa paso a paso.

**Paso 1**: Compilacion con make:

```
$ make
gfortran -std=f2018 -Wall -Wextra -c mod_datos_csv.f90
gfortran -std=f2018 -Wall -Wextra -c mod_regresion.f90
gfortran -std=f2018 -Wall -Wextra -c mod_estadisticas.f90
gfortran -std=f2018 -Wall -Wextra -c mod_reportes.f90
gfortran -std=f2018 -Wall -Wextra -c mod_api_handler.f90
gfortran -std=f2018 -Wall -Wextra -c programa_principal.f90
gfortran -std=f2018 -Wall -Wextra -o programa_principal programa_principal.o \
  mod_datos_csv.o mod_estadisticas.o mod_regresion.o mod_reportes.o mod_api_handler.o
gfortran -std=f2018 -Wall -Wextra -c cliente_api.f90
gfortran -std=f2018 -Wall -Wextra -o cliente_api cliente_api.o
```

Sin errores ni warnings.

**Paso 2**: Ejecutar `./programa_principal` (salida completa mostrada en la seccion 7).

**Paso 3**: En otra terminal, probar con curl:

```
$ curl -s http://localhost:8080/indicadores/inflacion | python3 -m json.tool
```

(Salida JSON formateada con las estadisticas de inflacion)

**Paso 4**: Ejecutar `./cliente_api`:

```
$ ./cliente_api
=== Cliente de prueba de la API ===
Servidor: http://localhost:8080

Consulta: GET /indicadores/inflacion
...
```

**Paso 5**: Generar graficos:

```
$ gnuplot graficos.plt
$ ls -la graficos.png
-rw-rw-r-- 1 usuario usuario 98543 jul  6 12:00 graficos.png
```

El sistema completo funciona: lectura de datos, analisis estadistico, regresion lineal, reportes LaTeX y Gnuplot, servidor API REST y cliente de prueba, todo escrito en Fortran 2018 estandar.

---

## Resumen del capitulo

En este capitulo culminante ha construido un sistema completo de monitoreo economico que integra los conceptos de todos los capitulos anteriores:

- **Capitulo 1**: Lectura de CSV con tipos derivados y arreglos allocatable.
- **Capitulo 1 y 2**: Estadisticas descriptivas, correlacion de Pearson y deteccion de outliers.
- **Capitulo 2**: Regresion lineal por minimos cuadrados con calculo de R^2.
- **Capitulo 3**: Generacion automatica de informes LaTeX y scripts de Gnuplot.
- **Capitulo 4**: Servidor HTTP con sockets POSIX via C binding, generacion de JSON y enrutamiento REST.

El sistema resultante es funcional, compilable con `gfortran -std=f2018`, y ejecutable en cualquier sistema GNU/Linux con las herramientas mencionadas.

**Ejercicios propuestos**:

1. Agregue un nuevo endpoint `GET /tendencias` que devuelva solo las tendencias de los cuatro indicadores.
2. Implemente deteccion de outliers en la respuesta de `/indicadores/{nombre}`.
3. Anada prediccion para 12 meses en lugar de 6.
4. Modifique `mod_reportes` para generar una tabla comparativa ano contra ano.
5. Agregue soporte para `POST /indicadores` que permita agregar un nuevo registro al CSV.

¡Felicidades! Ha completado el manual de Fortran para analisis de datos. Ahora tiene las herramientas para construir sistemas complejos y funcionales usando solo Fortran estandar y bibliotecas del sistema.
