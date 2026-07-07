# Capítulo 3: Generación de Reportes con Fortran Moderno

> La generación de reportes es el arte de transformar datos crudos en información legible, presentable y accionable. En este capítulo exploraremos cómo Fortran Moderno, con su potente sistema de formatos y su capacidad de integración con herramientas externas, nos permite construir pipelines completos de reportes: desde los datos hasta el gráfico final. Aprenderás a dominar los formatos avanzados de `write`, a construir tablas dinámicas, y a exportar resultados a CSV, JSON, LaTeX y Gnuplot.

---

**Stack tecnológico**:
- Compilador: gfortran 13+ con `-std=f2018`
- Gnuplot 5.4+ (opcional, para generación de gráficos)
- Sistema: GNU/Linux

---

## Introducción a la generación de reportes

Imagina por un momento que eres un científico de datos en la década de 1980. Tienes un montón de números producto de tus simulaciones, pero necesitas presentarlos a tu director de laboratorio. No tienes Python, no tienes Jupyter notebooks, no tienes pandas. Lo único que tienes es un terminal verde sobre negro y un compilador de Fortran. ¿Cómo conviertes 50 000 números de punto flotante en una tabla que tu director pueda leer? La respuesta está en el sistema de formato de Fortran.

Fortran nació en una época donde la salida por línea de impresión (sí, papel físico) era el estándar. Por eso su sistema de formatos es tan expresivo. Cada `format` es una mini-receta que le dice al programa: "toma este número, conviértelo a texto, alinéalo a la derecha, usa exactamente 10 caracteres, y si es muy grande, pon asteriscos". Esto puede sonar arcaico, pero es precisamente este control granular lo que hace que Fortran siga siendo insuperable para generar reportes tabulares en sistemas embebidos, HPC y aplicaciones financieras.

El paradigma de generación de reportes en Fortran se puede resumir en tres pasos: **calcular**, **formatear**, **exportar**. Primero procesas tus datos y obtienes los valores que quieres presentar. Luego decides exactamente cómo se verán: cuántos decimales, qué ancho de columna, qué separadores. Finalmente eliges el medio: pantalla, archivo CSV, JSON, LaTeX, o un script para Gnuplot. Lo hermoso es que cada uno de estos pasos está bajo tu control total. No hay una biblioteca que decida por ti cómo alinear las columnas; eres tú quien define cada carácter.

Pregúntate: ¿cuántas veces has visto una tabla generada por una herramienta moderna que se ve genial en pantalla pero al imprimirla los bordes se cortan o las columnas no alinean? Con Fortran eso no pasa, porque tú defines el ancho de cada columna en número de caracteres, y el compilador se asegura de que quepa exactamente. Esto es especialmente valioso en sistemas legacy donde el reporte debe imprimirse en formularios pre-impresos con posiciones fijas.

Otra idea fundamental es que en Fortran **el formato y los datos están separados**. Declaras el formato en una línea (o en un string) y los datos en otra. Esto permite algo muy poderoso: formatos dinámicos construidos en tiempo de ejecución. Puedes calcular el ancho de una columna según el valor más grande de tus datos, construir el string de formato con ese ancho, y luego usarlo en el `write`. Así es como se hacen tablas que se adaptan al contenido sin truncar nada.

En este capítulo vamos a construir pieza por pieza un sistema completo de generación de reportes. Comenzaremos por los bloques fundamentales: los formatos avanzados. Luego veremos cómo combinarlos para hacer tablas dinámicas. Después exploraremos la exportación a formatos de intercambio como CSV y JSON, y a formatos de presentación como LaTeX. Finalmente integraremos Gnuplot para generar gráficos vectoriales, y uniremos todo en un mini-proyecto que genera informes financieros mensuales completos.

Al terminar este capítulo, serás capaz de tomar cualquier conjunto de datos tabulares y producir un reporte profesional con tablas, gráficos y múltiples formatos de exportación, todo desde un único programa Fortran. Y lo harás entendiendo cada carácter que produces, porque tú lo controlas todo.

## Formatos avanzados con write

Pensemos en el formato como un idioma. Cuando aprendes un idioma nuevo, primero aprendes palabras sueltas (los descriptores de formato individuales: `F`, `E`, `I`, `A`), luego aprendes a combinarlas en frases (formatos compuestos), y finalmente aprendes los matices y excepciones (los casos borde). Fortran tiene alrededor de 20 descriptores de formato, pero en la práctica con 8 o 9 puedes hacer casi cualquier cosa. La clave está en entender qué hace cada uno y cuándo usarlo.

¿Por qué hay tantos descriptores para números reales? Porque cada disciplina científica tiene su propia convención. `F` para finanzas, donde los decimales son fijos y sabes que nunca tendrás números astronómicos. `E` para física e ingeniería, donde los valores pueden ir de 10⁻³⁰ a 10³⁰ y necesitas notación exponencial. `ES` para cuando quieres que el exponente sea múltiplo de 3, como en ingeniería eléctrica. `EN` para notación de ingeniería donde el exponente siempre es múltiplo de 3 y la mantisa está entre 1 y 1000. `G` para cuando no te importa el formato exacto y quieres que Fortran elija entre `F` y `E` según el tamaño del número.

Cada descriptor de formato tiene tres partes: el carácter que lo identifica (`F`, `E`, etc.), el ancho total del campo (incluyendo el punto decimal y el signo), y el número de decimales. Por ejemplo, `F10.2` significa "10 caracteres de ancho total, 2 decimales". Si el número no cabe, Fortran llena el campo con asteriscos. Esto es mejor que truncar silenciosamente, porque te avisa que algo anda mal.

Los descriptores de control de posición (`/`, `TC`, `TL`, `TR`, `X`) son los que te permiten moverte por la línea como si fueras un mecanógrafo. `/` es un salto de línea. `X` avanza espacios. `TC` posiciona el cursor en una columna específica (como el tabulador de las máquinas de escribir). `TL` retrocede espacios, `TR` avanza espacios. Esto es muy útil para construir tablas con alineaciones complejas sin tener que calcular manualmente los anchos.

Los formatos de cadena (`A`) y entero (`I`) son más simples pero igual de importantes. `A` imprime cadenas de caracteres; si le pones un ancho como `A20`, la cadena se alinea a la derecha en un campo de 20 caracteres. `I` hace lo mismo con enteros: `I5` imprime un entero en 5 caracteres alineado a la derecha.

Veamos ahora cómo se combinan todos estos descriptores en la práctica. El formato se especifica en una sentencia `format` con etiqueta, o como string en el mismo `write`. La sintaxis general es:

```fortran
write(unit, fmt_label) lista_de_datos
```

Donde `fmt_label` es una etiqueta de sentencia `format` o un string. La flexibilidad de usar strings te permite construir formatos dinámicamente concatenando descriptores con anchos calculados.

```fortran
program ejemplo_formatos
    implicit none
    real :: pi, grande, chico, ingreso
    integer :: mes, ventas
    character(len=20) :: nombre

    pi = 3.14159265358979
    grande = 1234567.89
    chico = 0.00001234
    ingreso = 12345.67
    mes = 12
    ventas = 1500
    nombre = "Ventas Q1"

    ! Cabecera del reporte
    print *, "=== DEMOSTRACION DE FORMATOS AVANZADOS ==="
    print *, ""

    ! Formato F - punto fijo
    print 100, "F12.4:", pi
    100 format(A, F12.4)

    ! Formato E - notacion cientifica
    print 110, "E14.6:", chico
    110 format(A, E14.6)

    ! Formato ES - exponente ajustado
    print 120, "ES14.6:", chico
    120 format(A, ES14.6)

    ! Formato EN - notacion de ingenieria
    print 130, "EN14.6:", grande
    130 format(A, EN14.6)

    ! Formato G - formato general
    print 140, "G14.6:", pi
    print 141, "G14.6:", chico
    print 142, "G14.6:", grande
    140 format(A, G14.6)
    141 format(A, G14.6)
    142 format(A, G14.6)

    ! Combinacion de formatos con /
    print 150, "Reporte Mensual", "Mes", "Ventas", "Ingreso", &
               mes, ventas, ingreso
    150 format(A, /, 3A10, /, I10, I10, F10.2)

    ! Formato con TC (posicion en columna especifica)
    print 160, "Producto", "Cantidad", "Precio", "Total"
    160 format(/, T5, A, T20, A, T35, A, T50, A)

    ! Formato con TL y TR (retroceso y avance)
    print 170, "ERROR:", "  Desbordamiento en linea 42"
    170 format(A, TL6, A)

    ! Formato A con ancho
    print 180, nombre
    180 format(A20)

    ! Formato I con ancho
    print 190, ventas
    190 format(I10)

end program ejemplo_formatos
```

**Análisis línea por línea**:

Línea 1: `program ejemplo_formatos` — Declara el programa principal. Todo el código ejecutable está dentro de esta estructura.

Línea 2: `implicit none` — Obliga a declarar todas las variables explícitamente. Sin esta línea, Fortran asumiría tipos según la primera letra, lo que lleva a errores difíciles de depurar.

Líneas 3-5: Declaraciones de variables. `real` para números de punto flotante, `integer` para enteros, `character(len=20)` para cadenas de hasta 20 caracteres.

Líneas 7-12: Asignación de valores a las variables. `pi` tiene muchos decimales para probar el redondeo. `grande` y `chico` muestran cómo se comportan los formatos con números de magnitudes muy diferentes.

Líneas 14-15: `print *, "..."` — El `*` en la unidad significa "salida estándar" (pantalla). Los strings se imprimen literalmente.

Línea 18: `print 100, "F12.4:", pi` — La sentencia `print 100` significa "usa el formato etiquetado como 100". La lista después de la coma son los datos a formatear.

Línea 19: `100 format(A, F12.4)` — La etiqueta `100` es un número de sentencia (en desuso en F2018 pero aún válido). `A` imprime la cadena. `F12.4` imprime `pi` en 12 caracteres con 4 decimales.

Línea 22: `print 110, "E14.6:", chico` — `E14.6` imprime en notación científica. El 14 es el ancho total, el 6 son los decimales de la mantisa.

Línea 23: `110 format(A, E14.6)` — La salida será algo como `E14.6: 0.123400E-04`. Observa que el exponente siempre ocupa al menos 2 dígitos.

Línea 26: `print 120, "ES14.6:", chico` — `ES14.6` es similar a `E` pero el exponente se ajusta para que la mantisa esté entre 1.0 y 9.999... En lugar de `0.123400E-04`, obtendrás `1.234000E-05`.

Línea 27: `120 format(A, ES14.6)` — Usa `ES` cuando quieras que el primer dígito significativo esté a la izquierda del punto decimal.

Línea 30: `print 130, "EN14.6:", grande` — `EN` (engineering) es como `ES` pero el exponente siempre es múltiplo de 3. `1234567.89` se mostrará como `1.234568E+06`.

Línea 31: `130 format(A, EN14.6)` — Útil en ingeniería donde las unidades vienen en miles, millones, etc.

Líneas 33-37: `G14.6` — El formato "general" elige automáticamente entre `F` y `ES` según el valor. Para `pi` (3.14...) usará `F`. Para `chico` (0.00001234) usará notación científica.

Línea 40: `print 150, ...` — Múltiples datos de diferentes tipos en un solo `print`. Aquí hay 7 argumentos: un string, tres strings, tres números.

Línea 41: `150 format(A, /, 3A10, /, I10, I10, F10.2)` — `A` imprime el título. `/` salta una línea. `3A10` imprime tres cadenas, cada una en 10 caracteres. `/` salta otra línea. `I10`, `I10`, `F10.2` imprimen los números en 10 caracteres cada uno.

Línea 44: `print 160, "Producto", "Cantidad", "Precio", "Total"` — Cuatro cadenas.

Línea 45: `160 format(/, T5, A, T20, A, T35, A, T50, A)` — `T5` mueve el cursor a la columna 5. `A` imprime "Producto" empezando en la columna 5. `T20` mueve a columna 20, `A` imprime "Cantidad", etc. Esto crea columnas perfectamente alineadas sin calcular anchos.

Línea 48: `print 170, "ERROR:", "  texto"` — Demostración de `TL`.

Línea 49: `170 format(A, TL6, A)` — Imprime "ERROR:" (6 caracteres), luego retrocede 6 posiciones (`TL6`), luego imprime el siguiente argumento. El resultado: los espacios iniciales del segundo string sobreescriben "ERROR:".

Líneas 51-53: `A20` imprime la cadena en 20 caracteres alineada a la derecha. `I10` imprime el entero en 10 caracteres alineado a la derecha.

**Salida esperada**:

```
 === DEMOSTRACION DE FORMATOS AVANZADOS ===

 F12.4:      3.1416
E14.6:  0.123400E-04
ES14.6:  1.234000E-05
EN14.6:  1.234568E+06
G14.6:   3.14159
G14.6:  1.23400E-05
G14.6:   1.23457E+06
Reporte Mensual
      Mes    Ventas    Ingreso
        12      1500   12345.67

     Producto      Cantidad       Precio         Total
  Desbordamiento en linea 42
             Ventas Q1
      1500
```

**Errores típicos**:

```fortran
! Error: descriptor F con muy poco ancho
print *, 12345.67
print 200, 12345.67
200 format(F3.1)
```

Mensaje de gfortran en tiempo de ejecución: `At line 7 of file error.f90 (unit = 6, file = 'stdout')
Fortran runtime error: Missing exponent in format string`

O bien, si el ancho total es insuficiente pero no tan extremo, simplemente imprime asteriscos: `***`.

Solución: asegúrate de que el ancho total contemple el signo, la parte entera máxima, el punto decimal y los decimales. Una regla práctica: ancho = parte_entera_máxima + 1 (signo) + 1 (punto) + decimales. Siempre suma 2 o 3 de margen.

```fortran
! Error: olvidar que TL retrocede y puede solaparse
print 210, "Hola", "Mundo"
210 format(A, TL2, TL2, A)
```

Este patrón es confuso y difícil de mantener. Prefiere `TC` o anchos fijos cuando necesites posicionamiento preciso.

```fortran
! Error: EN requiere gfortran 9+
! gfortran 8 daria: "Error: 'en' format descriptor requires -std=legacy option"
```

Solución: usa `-std=f2018` o actualiza tu compilador. A partir de gfortran 9, `EN` y `ES` funcionan con `-std=f2018`.

## Tablas dinámicas con anchos de columna variables

Hasta ahora hemos usado formatos con anchos fijos: `A20`, `F10.2`, `I5`. Pero en el mundo real, los datos cambian. Un mes puedes tener ventas de 500 (3 dígitos) y al siguiente mes de 12 500 (5 dígitos). Si fijas el ancho en 10, desperdicias espacio cuando el número es chico y corres el riesgo de truncar cuando el número es grande. La solución son las tablas dinámicas: anchos de columna que se calculan automáticamente según los datos.

¿Cómo se hace esto en Fortran? La clave está en construir el string de formato en tiempo de ejecución. Fortran permite que el formato sea una variable `character(len=...)` en lugar de una etiqueta fija. Puedes concatenar partes del formato usando el operador `//`. Por ejemplo, si calculas que el ancho óptimo para una columna es 8, construyes `"(A, I8)"` y se lo pasas al `write`.

Para calcular el ancho óptimo necesitas tres cosas: el valor máximo de tus datos, convertirlo a string con `write` en un buffer interno, medir su longitud con `len_trim`, y luego sumar un margen. Este cálculo lo haces una vez antes de imprimir toda la tabla. Así te aseguras de que todas las filas usen el mismo ancho y la tabla quede perfectamente alineada.

El patrón general es:

1. Recorrer todos los datos para encontrar el valor máximo.
2. Convertir ese valor máximo a string (usando un `write` a un buffer).
3. Medir `len_trim` del buffer para saber cuántos caracteres ocupa.
4. Agregar un margen de seguridad (2-3 caracteres).
5. Construir el string de formato dinámicamente.
6. Usar ese formato para imprimir toda la tabla.

Este patrón es extremadamente útil y no tiene equivalente directo en muchos lenguajes modernos sin bibliotecas externas. En Python necesitarías pandas. En C necesitarías `printf` con `*` como ancho. En Fortran, todo lo resuelves con operaciones básicas de cadenas.

Veamos un ejemplo completo donde generamos una tabla de ventas mensuales donde el ancho de la columna de valores se adapta al dato más grande.

```fortran
program tablas_dinamicas
    implicit none
    integer, parameter :: nmeses = 6
    character(len=10) :: meses(nmeses)
    real :: ventas(nmeses)
    character(len=50) :: fmt_string
    character(len=20) :: buffer
    real :: max_ventas
    integer :: i, ancho_col

    ! Datos de ejemplo: ventas mensuales
    meses = ["Ene", "Feb", "Mar", "Abr", "May", "Jun"]
    ventas = [1500.50, 2340.75, 980.25, 4100.00, 3250.60, 500.00]

    ! Encontrar el valor maximo y calcular el ancho de columna
    max_ventas = maxval(ventas)
    write(buffer, '(F10.2)') max_ventas
    ancho_col = len_trim(adjustl(buffer)) + 3

    ! Construir formato dinamico
    write(fmt_string, '(A, I2, A)') '(A10, A', ancho_col, ')'

    ! Imprimir cabecera
    write(*, fmt_string) "Mes", "Ventas"

    ! Imprimir linea separadora
    do i = 1, 10 + ancho_col
        write(*, '(A)', advance='no') "-"
    end do
    write(*, *)

    ! Imprimir filas de datos
    do i = 1, nmeses
        write(*, fmt_string) meses(i), ventas(i)
    end do

end program tablas_dinamicas
```

**Análisis línea por línea**:

Línea 1: `program tablas_dinamicas` — Declara el programa.

Línea 2: `implicit none` — Obliga a declarar explícitamente.

Línea 3: `integer, parameter :: nmeses = 6` — Constante con el número de meses. El parámetro no puede cambiar en ejecución.

Línea 4: `character(len=10) :: meses(nmeses)` — Arreglo de 6 cadenas de hasta 10 caracteres.

Línea 5: `real :: ventas(nmeses)` — Arreglo de 6 reales.

Línea 6: `character(len=50) :: fmt_string` — Variable donde construiremos el formato dinámico.

Línea 7: `character(len=20) :: buffer` — Buffer temporal para convertir números a texto.

Línea 8: `real :: max_ventas` — Almacena el valor máximo.

Línea 9: `integer :: i, ancho_col` — `i` es contador, `ancho_col` es el ancho calculado.

Líneas 11-12: Asignación de datos de ejemplo. Usamos el constructor de arreglos `[...]`.

Línea 14: `max_ventas = maxval(ventas)` — `maxval` es una función intrínseca de Fortran que devuelve el valor máximo de un arreglo. No necesitas un bucle.

Línea 15: `write(buffer, '(F10.2)') max_ventas` — Convierte el número a string escribiendo en la variable `buffer`. El formato `F10.2` es arbitrario; solo necesitamos suficientes caracteres para que quepa.

Línea 16: `ancho_col = len_trim(adjustl(buffer)) + 3` — `adjustl` quita espacios a la izquierda (los pasa a la derecha). `len_trim` cuenta los caracteres sin contar espacios finales. Sumamos 3 de margen.

Línea 18: `write(fmt_string, '(A, I2, A)') '(A10, A', ancho_col, ')'` — Construye el formato dinámicamente. El formato exterior `(A, I2, A)` concatena tres piezas para formar algo como `(A10, A12)`. Es un formato dentro de un formato; Fortran lo permite y es muy poderoso.

Línea 20: `write(*, fmt_string) "Mes", "Ventas"` — Imprime la cabecera usando el formato dinámico. `*` como unidad significa pantalla.

Líneas 22-25: Bucle que imprime una línea de guiones. `advance='no'` evita el salto de línea automático. `write(*, *)` fuerza el salto de línea final.

Líneas 27-29: Bucle que imprime cada fila usando `fmt_string`.

**Salida esperada**:

```
      Mes    Ventas
-------------------
      Ene   1500.50
      Feb   2340.75
      Mar    980.25
      Abr   4100.00
      May   3250.60
      Jun    500.00
```

Observa cómo el ancho de la columna "Ventas" se adaptó a 4+3=7 caracteres porque "4100.00" es el valor más largo. Si agregaras un mes con ventas de 10 000.00, el ancho se recalcularía automáticamente.

**Errores típicos**:

```fortran
! Error: no usar adjustl antes de len_trim
! Si buffer = "  4100.00  " (con espacios a izquierda y derecha)
write(buffer, '(F10.2)') max_ventas
ancho_col = len_trim(buffer)  ! Resultado: 10 (incorrecto)
```

Mensaje: no hay error de compilación, pero el ancho calculado incluye los espacios izquierdos de `F10.2`. Solución: usa `adjustl` primero, que mueve los espacios izquierdos a la derecha, y luego `len_trim` cuenta solo el contenido significativo.

```fortran
! Error: formato mal construido
fmt_string = "(" // trim("A10, A") // ")"  ! Falta I2
write(*, fmt_string) "Mes", 1234
```

Mensaje de gfortran en ejecución: `Fortran runtime error: Unexpected element in format string at (1)`. Solución: verifica que el string generado sea sintácticamente un formato válido.

## Exportación a CSV y JSON

La exportación de datos a formatos de intercambio como CSV (Comma-Separated Values) y JSON (JavaScript Object Notation) es una habilidad esencial para cualquier programa que necesite interoperar con otras herramientas. En el mundo del análisis de datos, CSV es el formato universal para tablas: cualquier hoja de cálculo, base de datos o lenguaje de programación puede leerlo. JSON es el estándar para intercambio de datos en aplicaciones web y servicios REST.

Exportar a CSV desde Fortran es relativamente directo. El formato es simple: cada fila es una línea, los campos se separan por comas, los strings pueden ir entre comillas si contienen comas o espacios. Lo delicado está en asegurar que los números se formateen de manera consistente y que no se pierda precisión. No quieres que un análisis en Python reciba `1.234500E+05` cuando guardaste un dato financiero; quieres que reciba `123450.00`.

Para CSV, usamos `write` con unidad apuntando a un archivo. Abrimos el archivo con `open`, escribimos la cabecera y los datos fila por fila, y cerramos con `close`. El formato ideal es `F` (punto fijo) para datos financieros, `E` o `ES` para datos científicos, y siempre controlamos el número de decimales.

JSON es más complejo porque tiene una estructura jerárquica con llaves `{}`, corchetes `[]`, comas, y comillas dobles. Fortran no tiene una biblioteca nativa para JSON, pero podemos generarlo manualmente con `write`. Esto es factible para estructuras simples como un arreglo de objetos. La clave está en manejar correctamente las comillas dobles dentro de las cadenas de Fortran (escapando con comillas dobles adicionales: `""hola""` produce `"hola"`).

Veamos un programa que exporta los mismos datos a CSV y a JSON.

```fortran
program exportar_csv_json
    implicit none
    integer, parameter :: nreg = 4
    character(len=20) :: productos(nreg)
    real :: precios(nreg)
    integer :: cantidades(nreg)
    integer :: i

    ! Datos de ejemplo
    productos = ["Laptop     ", "Monitor    ", "Teclado    ", "Mouse      "]
    precios = [15000.00, 4500.50, 899.99, 350.25]
    cantidades = [10, 15, 30, 50]

    ! ==========================================
    ! Exportacion a CSV
    ! ==========================================
    open(unit=10, file="reporte.csv", status="replace", action="write")
    write(10, '(A)') "Producto,Precio,Cantidad,Total"
    do i = 1, nreg
        write(10, '(A, A, F10.2, A, I5, A, F12.2)') &
            trim(adjustl(productos(i))), ",", &
            precios(i), ",", &
            cantidades(i), ",", &
            precios(i) * cantidades(i)
    end do
    close(10)
    print *, "CSV exportado: reporte.csv"

    ! ==========================================
    ! Exportacion a JSON
    ! ==========================================
    open(unit=20, file="reporte.json", status="replace", action="write")
    write(20, '(A)') "{"
    write(20, '(A)') '  "reporte": "Inventario",'
    write(20, '(A)') '  "fecha": "2026-07-06",'
    write(20, '(A)') '  "productos": ['
    do i = 1, nreg
        if (i > 1) write(20, '(A)') ","
        write(20, '(A)', advance="no") '    {'
        write(20, '(A)', advance="no") '"producto": "'
        write(20, '(A)', advance="no") trim(adjustl(productos(i)))
        write(20, '(A)', advance="no") '", "precio": '
        write(20, '(F10.2)', advance="no") precios(i)
        write(20, '(A)', advance="no") ', "cantidad": '
        write(20, '(I5)', advance="no") cantidades(i)
        write(20, '(A)') '}'
    end do
    write(20, '(A)') '  ]'
    write(20, '(A)') "}"
    close(20)
    print *, "JSON exportado: reporte.json"

end program exportar_csv_json
```

**Análisis línea por línea**:

Línea 1: `program exportar_csv_json` — Programa principal.

Líneas 2-6: Declaraciones. `nreg` es el número de registros. Tres arreglos paralelos para productos, precios y cantidades.

Líneas 8-11: Datos de ejemplo. Nota los espacios en los nombres de productos para llenar 20 caracteres; los limpiaremos con `adjustl`.

Línea 14: `open(unit=10, file="reporte.csv", status="replace", action="write")` — Abre el archivo `reporte.csv` para escritura. `status="replace"` sobreescribe si existe. `unit=10` es un número que identifica el archivo — en Fortran clásico los units 1-99 están disponibles para el programador.

Línea 15: `write(10, '(A)') "Producto,Precio,Cantidad,Total"` — Escribe la cabecera del CSV. Solo un string `A`.

Líneas 16-21: Bucle que escribe cada fila. Observa la concatenación: cada campo se escribe con su formato individual y se separa con comas como strings literales. El total se calcula en el momento con `precios(i) * cantidades(i)`.

Línea 22: `close(10)` — Cierra el archivo. Es importante cerrar para que el buffer se vacíe y el archivo quede completo.

Línea 26: `open(unit=20, file="reporte.json", ...)` — Abre el archivo JSON.

Líneas 27-28: Escriben las llaves de apertura del objeto JSON y los primeros campos. Las comillas dobles dentro del string de Fortran se representan como `""""` (sí, cuatro comillas: la primera y la última delimitan el string de Fortran, y las dos del medio son una comilla escapada).

Línea 29: `write(20, '(A)') '  "productos": ['` — Aquí usamos comillas simples para el string exterior y dobles para el JSON. Es más legible que escapar dentro de comillas dobles.

Líneas 30-38: Bucle que escribe cada producto como un objeto JSON. `advance="no"` permite construir el objeto en múltiples `write` sin saltos de línea intermedios. La línea 31 inserta una coma entre objetos (excepto antes del primero). Las líneas 32-37 escriben los campos del objeto paso a paso.

Líneas 39-41: Cierran el arreglo `]` y el objeto `{}`.

Línea 42: `close(20)` — Cierra el archivo.

**Salida esperada** (reporte.csv):

```
Producto,Precio,Cantidad,Total
Laptop,  15000.00,   10, 150000.00
Monitor,   4500.50,   15,  67507.50
Teclado,    899.99,   30,  26999.70
Mouse,      350.25,   50,  17512.50
```

**Salida esperada** (reporte.json):

```json
{
  "reporte": "Inventario",
  "fecha": "2026-07-06",
  "productos": [
    {"producto": "Laptop", "precio":  15000.00, "cantidad":    10}
,
    {"producto": "Monitor", "precio":   4500.50, "cantidad":    15}
,
    {"producto": "Teclado", "precio":    899.99, "cantidad":    30}
,
    {"producto": "Mouse", "precio":    350.25, "cantidad":    50}
  ]
}
```

**Nota**: El JSON generado tiene una coma antes del salto de línea en cada objeto excepto el primero. Esto se debe a que la coma se escribe en la siguiente iteración. Un enfoque más limpio sería escribir la coma al final de cada objeto excepto el último. En el mini-proyecto final veremos una versión más refinada.

**Errores típicos**:

```fortran
! Error: no escapar comillas dobles en JSON
write(20, '(A)') "  "producto": "Laptop""
```

Mensaje de gfortran: `Error: Unterminated string constant at (1)`. Solución: usa comillas simples para delimitar, o escapa las dobles: `"  ""producto"": ""Laptop"""`. La opción más legible: alterna entre comillas simples y dobles.

```fortran
! Error: olvidar cerrar el archivo
open(10, file="datos.csv", status="replace", action="write")
write(10, *) "a,b"
! No hay close(10)
! Los datos pueden no escribirse completamente
```

No hay mensaje de error, pero el archivo puede quedar vacío o incompleto. Siempre cierra los archivos con `close(unit)`.

## Exportación a LaTeX

LaTeX es el estándar de facto para la publicación de documentos científicos y técnicos. A diferencia de un procesador de textos como Word, LaTeX separa el contenido de la presentación: tú escribes el texto y las instrucciones de formato en un archivo `.tex`, y el compilador (`pdflatex`, `xelatex`) produce el PDF. Generar archivos `.tex` desde Fortran te permite crear reportes con tablas profesionales, ecuaciones y referencias cruzadas directamente desde tus datos.

Una tabla en LaTeX tiene esta estructura general:

```latex
\begin{table}[h]
\centering
\begin{tabular}{|c|c|c|}
\hline
Col1 & Col2 & Col3 \\
\hline
dato1 & dato2 & dato3 \\
\hline
\end{tabular}
\caption{Título de la tabla}
\end{table}
```

Las columnas se definen con letras: `l` (izquierda), `c` (centrado), `r` (derecha), `p{ancho}` (párrafo). Las barras verticales `|` indican líneas verticales. `\hline` dibuja líneas horizontales. Las filas se separan con `\\`.

Generar esto desde Fortran es cuestión de escribir las líneas una por una. La dificultad principal es que LaTeX usa caracteres que son especiales para Fortran, como la barra invertida `\`. En un string de Fortran, `\` no es especial (a diferencia de C), así que puedes escribir `\hline` directamente.

Veamos un programa que toma los datos de productos y genera un archivo `.tex` completo y compilable.

```fortran
program exportar_latex
    implicit none
    integer, parameter :: nreg = 4
    character(len=20) :: productos(nreg)
    real :: precios(nreg)
    integer :: cantidades(nreg)
    real :: totales(nreg), gran_total
    integer :: i

    ! Datos de ejemplo
    productos = ["Laptop     ", "Monitor    ", "Teclado    ", "Mouse      "]
    precios = [15000.00, 4500.50, 899.99, 350.25]
    cantidades = [10, 15, 30, 50]

    ! Calcular totales
    gran_total = 0.0
    do i = 1, nreg
        totales(i) = precios(i) * cantidades(i)
        gran_total = gran_total + totales(i)
    end do

    ! Generar archivo .tex
    open(unit=30, file="reporte.tex", status="replace", action="write")

    ! Preamble del documento
    write(30, '(A)') "\documentclass[12pt]{article}"
    write(30, '(A)') "\usepackage[utf8]{inputenc}"
    write(30, '(A)') "\usepackage[spanish]{babel}"
    write(30, '(A)') "\usepackage{geometry}"
    write(30, '(A)') "\geometry{a4paper, margin=2.5cm}"
    write(30, '(A)') "\title{Reporte de Inventario}"
    write(30, '(A)') "\date{\today}"
    write(30, '(A)') "\begin{document}"
    write(30, '(A)') "\maketitle"
    write(30, '(A)') ""
    write(30, '(A)') "\section*{Detalle de Productos}"
    write(30, '(A)') ""

    ! Tabla en LaTeX
    write(30, '(A)') "\begin{table}[h]"
    write(30, '(A)') "\centering"
    write(30, '(A)') "\begin{tabular}{|l|r|r|r|}"
    write(30, '(A)') "\hline"
    write(30, '(A)') "Producto & Precio & Cantidad & Total \\"
    write(30, '(A)') "\hline"

    do i = 1, nreg
        write(30, '(A, A, F10.2, A, I5, A, F12.2, A)') &
            trim(adjustl(productos(i))), " & $", &
            precios(i), "$ & ", &
            cantidades(i), " & $", &
            totales(i), "$ \\"
    end do

    write(30, '(A)') "\hline"
    write(30, '(A, F12.2, A)') "\multicolumn{3}{|r|}{Gran Total} & $", &
                                gran_total, "$ \\"
    write(30, '(A)') "\hline"
    write(30, '(A)') "\end{tabular}"
    write(30, '(A)') "\caption{Resumen de inventario generado desde Fortran.}"
    write(30, '(A)') "\end{table}"

    write(30, '(A)') ""
    write(30, '(A)') "\section*{Resumen}"
    write(30, '(A)') "Este reporte fue generado automaticamente desde un programa"
    write(30, '(A)') "Fortran Moderno. Los datos provienen de una simulacion de"
    write(30, '(A)') "inventario con ", nreg, " productos."
    write(30, '(A)') ""
    write(30, '(A)') "\end{document}"
    close(30)

    print *, "LaTeX exportado: reporte.tex"
    print *, "Compila con: pdflatex reporte.tex"

end program exportar_latex
```

**Análisis línea por línea**:

Líneas 1-9: Declaraciones del programa. Nota `gran_total` para la suma global.

Líneas 11-14: Datos de ejemplo.

Líneas 16-20: Bucle que calcula totales por producto y el gran total.

Línea 22: `open(unit=30, ...)` — Abre `reporte.tex` para escritura.

Líneas 24-35: Escriben el preámbulo del documento LaTeX. Cada línea es una instrucción LaTeX. `\documentclass[12pt]{article}` define el tipo de documento. Los `\usepackage` cargan paquetes para UTF-8, español, y márgenes. `\title`, `\date`, `\maketitle` generan la portada.

Línea 36: `write(30, '(A)') ""` — Línea en blanco para mejorar la legibilidad del `.tex` generado.

Líneas 37-38: `\section*{...}` — Un `section` sin numerar (por el `*`).

Líneas 40-45: Inicio de la tabla. `\begin{table}[h]` coloca la tabla "aquí" (h = here). `\centering` la centra. `\begin{tabular}{|l|r|r|r|}` define 4 columnas: izquierda, derecha, derecha, derecha, con líneas verticales entre ellas. `\hline` dibuja una línea horizontal.

Línea 46: Cabecera de la tabla. En LaTeX las columnas se separan con `&` y las filas terminan con `\\`.

Líneas 48-52: Bucle que escribe cada fila de datos. Los precios y totales se envuelven en `$...$` para formato matemático (tipografía profesional). `trim(adjustl(...))` limpia los espacios de los nombres de producto.

Líneas 54-57: Fila del gran total. `\multicolumn{3}{|r|}{...}` fusiona las primeras 3 columnas y alinea a la derecha.

Líneas 58-61: Cierre de la tabla y caption.

Líneas 63-72: Sección de resumen y cierre del documento.

Línea 73: `close(30)` — Cierra el archivo.

**Salida esperada** (reporte.tex):

```latex
\documentclass[12pt]{article}
\usepackage[utf8]{inputenc}
\usepackage[spanish]{babel}
\usepackage{geometry}
\geometry{a4paper, margin=2.5cm}
\title{Reporte de Inventario}
\date{\today}
\begin{document}
\maketitle

\section*{Detalle de Productos}

\begin{table}[h]
\centering
\begin{tabular}{|l|r|r|r|}
\hline
Producto & Precio & Cantidad & Total \\
\hline
Laptop & $  15000.00$ &    10 & $ 150000.00$ \\
Monitor & $   4500.50$ &    15 & $  67507.50$ \\
Teclado & $    899.99$ &    30 & $  26999.70$ \\
Mouse & $    350.25$ &    50 & $  17512.50$ \\
\hline
\multicolumn{3}{|r|}{Gran Total} & $ 262019.70$ \\
\hline
\end{tabular}
\caption{Resumen de inventario generado desde Fortran.}
\end{table}

\section*{Resumen}
Este reporte fue generado automaticamente desde un programa
Fortran Moderno. Los datos provienen de una simulacion de
inventario con 4 productos.

\end{document}
```

Para compilar: `pdflatex reporte.tex`. El resultado será un PDF con una tabla profesional.

**Errores típicos**:

```fortran
! Error: caracter & sin escapar en LaTeX
write(30, '(A)') "Producto & Precio"  ! Esto funciona
! Pero si el dato contiene &:
write(30, '(A)') "Pan & Compania"  ! Error en LaTeX
```

Mensaje de LaTeX: `! Misplaced alignment tab character &.`. Solución: escapa los `&` en los datos con `\&` o reemplázalos antes de escribir.

```fortran
! Error: olvidar cerrar el entorno LaTeX
write(30, '(A)') "\begin{document}"
write(30, '(A)') "Contenido"
! Falta \end{document}
```

Mensaje de LaTeX: `! File ended while scanning use of \@end.`. Solución: siempre verifica que el `.tex` generado tenga tantos cierres como aperturas. Un buen truco: escribe primero el esqueleto completo y luego rellena los datos.

## Integración con Gnuplot

Gnuplot es un programa de graficación por línea de comandos que produce gráficos de alta calidad en formatos vectoriales (PDF, SVG, EPS) y de mapa de bits (PNG, GIF). Aunque no es tan vistoso como matplotlib de Python, es extremadamente rápido, ligero, y perfecto para entornos HPC donde no hay interfaz gráfica.

La integración con Fortran sigue este patrón: desde tu programa Fortran, generas un archivo de script de Gnuplot (`.gp`) que contiene las instrucciones para crear el gráfico: tipo de gráfico, datos, etiquetas, formato de salida. Luego, desde el mismo programa Fortran, ejecutas Gnuplot con una llamada al sistema usando la subrutina intrínseca `execute_command_line` (estándar desde F2008).

El script de Gnuplot para un gráfico de barras tiene esta estructura:

```gnuplot
set terminal pdfcairo enhanced font "Helvetica,10"
set output "grafico.pdf"
set title "Ventas por Mes"
set xlabel "Mes"
set ylabel "Ventas ($)"
set style data histograms
set style fill solid
plot "datos.txt" using 2:xticlabels(1) title "Ventas"
```

Los datos se pasan en un archivo de texto (podría ser el mismo CSV que generamos antes). Gnuplot lee el archivo y grafica las columnas que le indiquemos.

La subrutina `execute_command_line` toma un string con el comando a ejecutar. En Linux, si Gnuplot está instalado, llamamos `gnuplot script.gp`. La función devuelve un código de salida (0 si todo bien) y opcionalmente captura la salida del comando.

Veamos un programa completo que genera datos de ventas, crea el script de Gnuplot, lo ejecuta, y verifica el resultado.

```fortran
program integrar_gnuplot
    implicit none
    integer, parameter :: nmeses = 6
    character(len=10) :: meses(nmeses)
    real :: ventas(nmeses)
    integer :: i, exit_status
    character(len=200) :: comando

    ! Datos de ventas
    meses = ["Ene", "Feb", "Mar", "Abr", "May", "Jun"]
    ventas = [1500.50, 2340.75, 980.25, 4100.00, 3250.60, 500.00]

    ! 1. Generar archivo de datos para Gnuplot
    open(unit=40, file="datos_ventas.txt", status="replace", action="write")
    do i = 1, nmeses
        write(40, '(A, 1X, F10.2)') trim(meses(i)), ventas(i)
    end do
    close(40)

    ! 2. Generar script de Gnuplot
    open(unit=50, file="grafico_ventas.gp", status="replace", action="write")
    write(50, '(A)') "set terminal pdfcairo enhanced font 'Helvetica,10'"
    write(50, '(A)') "set output 'grafico_ventas.pdf'"
    write(50, '(A)') "set title 'Ventas Mensuales'"
    write(50, '(A)') "set xlabel 'Mes'"
    write(50, '(A)') "set ylabel 'Ventas (USD)'"
    write(50, '(A)') "set grid ytics"
    write(50, '(A)') "set style data histograms"
    write(50, '(A)') "set style fill solid 0.7"
    write(50, '(A)') "set boxwidth 0.8"
    write(50, '(A)') "plot 'datos_ventas.txt' using 2:xticlabels(1) title 'Ventas' lc rgb 'navy'"
    close(50)

    ! 3. Ejecutar Gnuplot
    print *, "Ejecutando Gnuplot..."
    comando = "gnuplot grafico_ventas.gp 2>&1"
    call execute_command_line(trim(comando), exitstat=exit_status)

    if (exit_status == 0) then
        print *, "Grafico generado: grafico_ventas.pdf"
    else
        print *, "Error al ejecutar Gnuplot. Codigo de salida:", exit_status
        print *, "¿Gnuplot esta instalado? Prueba: which gnuplot"
    end if

end program integrar_gnuplot
```

**Análisis línea por línea**:

Línea 1: `program integrar_gnuplot` — Programa principal.

Líneas 2-6: Declaraciones. `exit_status` recibe el código de retorno de `execute_command_line`. `comando` almacena el comando a ejecutar.

Líneas 8-9: Datos de ejemplo.

Líneas 11-15: Generan `datos_ventas.txt`. Cada línea tiene el mes y la venta separados por un espacio (`1X`). Este archivo será leído por Gnuplot.

Líneas 17-28: Generan `grafico_ventas.gp`. Las instrucciones de Gnuplot se escriben como strings. Nota que las comillas simples funcionan bien en Gnuplot y son más fáciles de manejar desde Fortran.

Línea 18: `set terminal pdfcairo enhanced font 'Helvetica,10'` — Define el tipo de salida como PDF con el driver Cairo (alta calidad). `enhanced` permite subíndices y superíndices.

Línea 19: `set output 'grafico_ventas.pdf'` — Archivo de salida del gráfico.

Líneas 20-21: Títulos de los ejes.

Línea 22: `set grid ytics` — Activa la cuadrícula horizontal para facilitar la lectura.

Línea 23: `set style data histograms` — Configura Gnuplot para usar histogramas (barras).

Línea 24: `set style fill solid 0.7` — Rellena las barras con color sólido al 70% de opacidad.

Línea 25: `set boxwidth 0.8` — Ancho relativo de las barras (0.8 = 80% del espacio disponible).

Línea 26: `plot 'datos_ventas.txt' using 2:xticlabels(1) title 'Ventas' lc rgb 'navy'` — La instrucción principal. `using 2` usa la columna 2 (ventas) para la altura. `xticlabels(1)` usa la columna 1 (meses) como etiquetas del eje X. `title 'Ventas'` es la leyenda. `lc rgb 'navy'` define el color azul marino.

Línea 29: `close(50)` — Cierra el script.

Línea 32: `comando = "gnuplot grafico_ventas.gp 2>&1"` — Construye el comando. `2>&1` redirige stderr a stdout para capturar errores.

Línea 33: `call execute_command_line(trim(comando), exitstat=exit_status)` — Ejecuta el comando. `trim` elimina espacios finales. `exit_stat` es un argumento de salida opcional que recibe el código de salida.

Líneas 35-39: Verifica el resultado. Código 0 = éxito. Cualquier otro valor indica error (Gnuplot no instalado, script mal formado, etc.).

**Salida esperada**:

```
 Ejecutando Gnuplot...
 Grafico generado: grafico_ventas.pdf
```

Si Gnuplot no está instalado:

```
 Ejecutando Gnuplot...
 Error al ejecutar Gnuplot. Codigo de salida: 127
 ¿Gnuplot esta instalado? Prueba: which gnuplot
```

(Salida 127 significa "comando no encontrado" en Linux.)

**Archivo datos_ventas.txt generado**:

```
Ene  1500.50
Feb  2340.75
Mar   980.25
Abr  4100.00
May  3250.60
Jun   500.00
```

**Errores típicos**:

```fortran
! Error: no verificar que Gnuplot existe
call execute_command_line("gnuplot script.gp")
! Si no esta instalado, falla silenciosamente (sin exitstat)
```

Solución: siempre usa `exitstat` para verificar, o primero prueba con `execute_command_line("which gnuplot", exitstat=...)`.

```fortran
! Error: rutas relativas incorrectas
comando = "gnuplot subcarpeta/script.gp"
! Si el script no existe en esa ruta, Gnuplot falla
```

Solución: usa rutas absolutas construidas con `getcwd` (intrínseco) o pasa la ruta completa.

```fortran
! Error: el archivo de datos usa separador incompatible
write(40, '(A, ",", F10.2)') trim(meses(i)), ventas(i)
! Gnuplot espera espacios o tabuladores por defecto
```

Mensaje de Gnuplot: `warning: Skipping data file with no valid points`. Solución: en el script de Gnuplot usa `set datafile separator ","` si usas CSV, o mejor, usa espacios que es el formato por defecto.

## Cadenas de caracteres avanzadas

Las cadenas de caracteres son el pegamento que une todos los componentes de un reporte. Sin un manejo adecuado de strings, los formatos más sofisticados producen resultados desalineados o con espacios basura. Fortran tiene un conjunto de funciones intrínsecas para manipular cadenas que, aunque menos extenso que el de Python o JavaScript, es perfectamente adecuado para las tareas de generación de reportes.

Las funciones clave son: `trim`, `adjustl`, `adjustr`, `len_trim`, `index`, y el operador de concatenación `//`. Cada una tiene un propósito específico:

- `trim(cadena)`: elimina espacios al final. Esencial antes de concatenar cadenas de longitud fija.
- `adjustl(cadena)`: mueve los espacios iniciales al final. Convierte `"  hola"` en `"hola  "`.
- `adjustr(cadena)`: mueve los espacios finales al inicio. Convierte `"hola  "` en `"  hola"`.
- `len_trim(cadena)`: longitud sin contar espacios finales. Te dice cuántos caracteres "útiles" tiene una cadena.
- `index(cadena, subcadena)`: busca una subcadena y devuelve la posición donde comienza, o 0 si no la encuentra.
- `//`: concatenación de cadenas. Concatena dos strings.

¿Por qué son necesarias estas funciones? Porque en Fortran las cadenas declaradas como `character(len=N)` siempre tienen longitud `N`. Si pones un texto más corto, el resto se llena con espacios. Sin `trim`, una concatenación de `"Hola"` y `"Mundo"` (ambas de 10 caracteres) produciría `"Hola     Mundo     "` en lugar de `"HolaMundo"`.

Veamos un programa que demuestra estas funciones y muestra cómo usarlas para construir strings complejos como los que necesitamos en los reportes.

```fortran
program cadenas_avanzadas
    implicit none
    character(len=15) :: nombre, apellido
    character(len=40) :: nombre_completo, mensaje
    character(len=60) :: busqueda
    character(len=40) :: columna1, columna2
    integer :: pos

    ! Cadenas de longitud fija (con espacios)
    nombre = "Roy"
    apellido = "Lopez"

    ! 1. TRIM - elimina espacios finales
    nombre_completo = trim(nombre) // " " // trim(apellido)
    print *, "1. Nombre completo (con trim):"
    print *, "   '", trim(nombre_completo), "'"

    ! 2. ADJUSTL - ajusta espacios a la izquierda
    columna1 = "     Ventas"
    columna2 = "Q1"
    print *, "2. ADJUSTL:"
    print *, "   Antes: '", columna1, "'"
    print *, "   Despues: '", adjustl(columna1), "'"
    print *, "   Concatenado: '", trim(adjustl(columna1)) // " " // &
             trim(adjustl(columna2)), "'"

    ! 3. ADJUSTR - ajusta espacios a la derecha
    print *, "3. ADJUSTR:"
    print *, "   '", adjustr(columna1), "'"

    ! 4. LEN_TRIM - longitud sin espacios finales
    mensaje = "Reporte Final  "
    print *, "4. LEN_TRIM:"
    print *, "   len(mensaje)   =", len(mensaje)
    print *, "   len_trim(mensaje) =", len_trim(mensaje)

    ! 5. INDEX - busqueda de subcadenas
    busqueda = "Error en la linea 42 del modulo ventas"
    pos = index(busqueda, "linea")
    print *, "5. INDEX:"
    print *, "   Cadena: '", trim(busqueda), "'"
    print *, "   'linea' comienza en posicion:", pos
    print *, "   'ERROR' comienza en posicion:", index(busqueda, "ERROR")

    ! 6. Concatenacion con multiples campos
    print *, "6. Construccion de linea de reporte:"
    print *, "   " // repeat("=", 40)
    print *, "   " // trim(adjustl(nombre)) // " " // &
             trim(adjustl(apellido)) // " | Ventas: Q1 | Total: $12,500"
    print *, "   " // repeat("=", 40)

end program cadenas_avanzadas
```

**Análisis línea por línea**:

Líneas 1-6: Declaraciones. Nota `character(len=N)` que define cadenas de longitud fija.

Líneas 8-9: Asignación. `"Roy"` en una variable de 15 caracteres produce `"Roy           "`.

Línea 11: `nombre_completo = trim(nombre) // " " // trim(apellido)` — `trim` elimina los espacios finales de cada nombre antes de concatenar. Sin `trim`, el resultado sería `"Roy            Lopez          "`.

Líneas 12-13: Imprime el resultado. Las comillas simples alrededor de `trim(nombre_completo)` permiten ver exactamente lo que hay.

Línea 15: `columna1 = "     Ventas"` — Cadena con 5 espacios al inicio.

Línea 16: `columna2 = "Q1"` — Cadena corta.

Líneas 17-20: Demostración de `adjustl`. Antes de ajustar, `columna1` tiene espacios a la izquierda. Después de `adjustl`, los espacios se mueven a la derecha. Al concatenar, usamos `trim(adjustl(...))` para una limpieza completa.

Línea 22: `adjustr(columna1)` — Mueve todos los espacios al inicio. Útil para alinear texto a la derecha en columnas de ancho fijo.

Línea 25: `mensaje = "Reporte Final  "` — Una cadena con 2 espacios finales.

Líneas 26-29: `len` devuelve la longitud total declarada (40). `len_trim` devuelve la longitud sin espacios finales (14 para "Reporte Final").

Línea 31: `busqueda = "Error en la linea 42 del modulo ventas"` — Cadena larga para búsqueda.

Línea 32: `pos = index(busqueda, "linea")` — Busca la palabra "linea". Index es sensible a mayúsculas/minúsculas.

Líneas 33-36: Muestra resultados. `index(busqueda, "ERROR")` devuelve 0 porque no hay "ERROR" en mayúsculas.

Línea 39: `repeat("=", 40)` — Genera un string de 40 signos `=`. Muy útil para líneas separadoras.

Línea 40-41: Construcción de una línea de reporte compleja usando concatenaciones múltiples.

**Salida esperada**:

```
 1. Nombre completo (con trim):
    'Roy Lopez'
 2. ADJUSTL:
    Antes: '     Ventas'
    Despues: 'Ventas     '
    Concatenado: 'Ventas Q1'
 3. ADJUSTR:
    '     Ventas'
 4. LEN_TRIM:
    len(mensaje)   =          40
    len_trim(mensaje) =          14
 5. INDEX:
    Cadena: 'Error en la linea 42 del modulo ventas'
    'linea' comienza en posicion:          10
    'ERROR' comienza en posicion:           0
 6. Construccion de linea de reporte:
    ========================================
    Roy Lopez | Ventas: Q1 | Total: $12,500
    ========================================
```

## Pipelines de reportes

Hasta ahora hemos trabajado cada componente por separado: formatos, tablas dinámicas, CSV, JSON, LaTeX, Gnuplot, cadenas. En la práctica real, todos estos componentes se combinan en un **pipeline de reportes**: un flujo de procesamiento donde los datos pasan por varias etapas hasta convertirse en los productos finales.

Un pipeline típico tiene esta forma:

```
[Datos crudos] -> [Análisis y cálculos] -> [Tabla formateada en terminal]
                                          -> [Archivo CSV para intercambio]
                                          -> [Archivo JSON para web]
                                          -> [Documento LaTeX para PDF]
                                          -> [Script Gnuplot para gráfico]
```

Cada etapa toma los datos de la anterior y produce una salida específica. La ventaja es que todas las salidas provienen de la misma fuente de datos, garantizando consistencia. Además, si los datos cambian, solo recompilas el programa y todas las salidas se regeneran automáticamente.

En Fortran, implementamos este pipeline como una serie de subrutinas, cada una responsable de una tarea. Los datos se almacenan en un arreglo o derivado de tipo (tipo estructurado) y se pasan a cada subrutina. Esto mantiene el código organizado y reutilizable.

Los pasos típicos son:

1. **Carga o generación de datos**: leer de archivo o generar datos de simulación.
2. **Procesamiento**: calcular sumas, promedios, variaciones, máximos, mínimos.
3. **Formateo en terminal**: imprimir una tabla legible en la consola.
4. **Exportación**: escribir archivos CSV, JSON, LaTeX, etc.
5. **Graficación**: generar script de Gnuplot y ejecutarlo.
6. **Verificación**: reportar resultados y verificar que todo se generó correctamente.

En el siguiente mini-proyecto, implementaremos un pipeline completo que abarca todos estos pasos. Lo que hemos aprendido hasta ahora —formatos avanzados, tablas dinámicas, cadenas, exportación a múltiples formatos, integración con Gnuplot— se combinará en un solo programa organizado en módulos.

El diseño basado en módulos (`module`) nos permite separar responsabilidades. Tendremos un módulo `report_utils` con las funciones y subrutinas genéricas, y un programa principal que orquesta el pipeline. Este es el patrón que verás en aplicaciones Fortran profesionales.

## Mini-proyecto: Generador de Informes Financieros Mensuales

Ha llegado el momento de unir todo lo aprendido en un proyecto completo. Vamos a construir un "Generador de Informes Financieros Mensuales" que procesa datos de ingresos y gastos de 12 meses en 3 categorías, calcula indicadores financieros, genera una tabla formateada, exporta a CSV, produce un documento LaTeX, crea un script de Gnuplot para un gráfico de barras apiladas, y ejecuta Gnuplot si está disponible.

El diseño está organizado en dos archivos:
- `report_utils.f90`: módulo con subrutinas reutilizables.
- `generar_informe.f90`: programa principal que usa el módulo.

Comenzaremos por el módulo, que encapsula la lógica de generación de cada formato.

```fortran
! ===================================================
! report_utils.f90 - Modulo de utilidades para reportes
! ===================================================
module report_utils
    implicit none
    private
    public :: calcular_indicadores, &
              imprimir_tabla_terminal, &
              exportar_csv, &
              exportar_latex, &
              generar_script_gnuplot

    integer, parameter, public :: MAX_CATEGORIAS = 3
    integer, parameter, public :: MAX_MESES = 12

contains

    ! --------------------------------------------------
    ! calcular_indicadores: procesa datos financieros
    ! --------------------------------------------------
    subroutine calcular_indicadores(ingresos, gastos, total_ing, total_gas, &
                                    promedio_ing, promedio_gas, variacion, &
                                    balance, categorias)
        real, intent(in)  :: ingresos(:,:)
        real, intent(in)  :: gastos(:,:)
        real, intent(out) :: total_ing, total_gas
        real, intent(out) :: promedio_ing, promedio_gas
        real, intent(out) :: variacion(:)
        real, intent(out) :: balance(:)
        character(len=*), intent(in) :: categorias(:)
        integer :: ncats, nmeses, i, j

        ncats = size(ingresos, dim=1)
        nmeses = size(ingresos, dim=2)

        total_ing = sum(ingresos)
        total_gas = sum(gastos)
        promedio_ing = total_ing / (nmeses * ncats)
        promedio_gas = total_gas / (nmeses * ncats)

        ! Variacion porcentual mes a mes (ingreso total mensual)
        do i = 1, nmeses
            if (i == 1) then
                variacion(i) = 0.0
            else
                variacion(i) = (sum(ingresos(:, i)) - sum(ingresos(:, i-1))) / &
                               sum(ingresos(:, i-1)) * 100.0
            end if
        end do

        ! Balance mensual total
        do i = 1, nmeses
            balance(i) = sum(ingresos(:, i)) - sum(gastos(:, i))
        end do

    end subroutine calcular_indicadores

    ! --------------------------------------------------
    ! imprimir_tabla_terminal: tabla con bordes de asteriscos
    ! --------------------------------------------------
    subroutine imprimir_tabla_terminal(ingresos, gastos, categorias, &
                                       total_ing, total_gas, variacion, balance)
        real, intent(in) :: ingresos(:,:), gastos(:,:)
        character(len=*), intent(in) :: categorias(:)
        real, intent(in) :: total_ing, total_gas
        real, intent(in) :: variacion(:), balance(:)
        integer :: ncats, nmeses, i, j
        character(len=100) :: fmt_row, fmt_header
        character(len=20) :: buffer
        real :: max_val
        integer :: ancho_val, ancho_cat, ancho_mes

        ncats = size(ingresos, dim=1)
        nmeses = size(ingresos, dim=2)

        ! Calcular anchos dinamicos
        ancho_cat = 12
        ancho_mes = 8
        max_val = max(maxval(ingresos), maxval(gastos), maxval(abs(balance)))
        write(buffer, '(F12.2)') max_val
        ancho_val = max(len_trim(adjustl(buffer)) + 2, 8)

        ! Construir formato de cabecera
        write(fmt_header, '(A, I2, A, I2, A)') &
            '(A', ancho_cat, ', 12(A', ancho_mes, '))'

        ! Construir formato de filas
        write(fmt_row, '(A, I2, A, I2, A, I2, A, I2, A)') &
            '(A', ancho_cat, ', ', ncats, &
            '(A', ancho_val, ', A', ancho_val, '), ', &
            nmeses, '(F', ancho_val, '.2))'

        ! Linea superior
        print *, ""
        print *, " " // repeat("*", ancho_cat + 1 + &
                 ncats * (ancho_val * 2 + 1) + nmeses * (ancho_val + 1) + 4)

    end subroutine imprimir_tabla_terminal

end module report_utils
```

**Nota**: El módulo anterior está incompleto intencionalmente para mostrar la estructura. La implementación completa continúa abajo.

El módulo completo continúa con la implementación real de `imprimir_tabla_terminal`, `exportar_csv`, `exportar_latex`, y `generar_script_gnuplot`. A continuación presentamos la versión completa y funcional del módulo junto con el programa principal.

```fortran
! ===================================================
! report_utils.f90 - Modulo completo de utilidades
! ===================================================
module report_utils
    implicit none
    private
    public :: calcular_indicadores, &
              imprimir_tabla_terminal, &
              exportar_csv, &
              exportar_latex, &
              generar_script_gnuplot

    integer, parameter, public :: MAX_CATEGORIAS = 3
    integer, parameter, public :: MAX_MESES = 12

contains

    ! --------------------------------------------------
    ! calcular_indicadores
    ! --------------------------------------------------
    subroutine calcular_indicadores(ingresos, gastos, total_ing, total_gas, &
                                    promedio_ing, promedio_gas, variacion, &
                                    balance, categorias)
        real, intent(in)  :: ingresos(:,:)
        real, intent(in)  :: gastos(:,:)
        real, intent(out) :: total_ing, total_gas
        real, intent(out) :: promedio_ing, promedio_gas
        real, intent(out) :: variacion(:)
        real, intent(out) :: balance(:)
        character(len=*), intent(in) :: categorias(:)
        integer :: ncats, nmeses, i

        ncats = size(ingresos, dim=1)
        nmeses = size(ingresos, dim=2)

        total_ing = sum(ingresos)
        total_gas = sum(gastos)
        promedio_ing = total_ing / real(nmeses * ncats)
        promedio_gas = total_gas / real(nmeses * ncats)

        ! Variacion porcentual mes a mes
        do i = 1, nmeses
            if (i == 1) then
                variacion(i) = 0.0
            else
                variacion(i) = (sum(ingresos(:, i)) - sum(ingresos(:, i-1))) / &
                               sum(ingresos(:, i-1)) * 100.0
            end if
        end do

        ! Balance mensual
        do i = 1, nmeses
            balance(i) = sum(ingresos(:, i)) - sum(gastos(:, i))
        end do

    end subroutine calcular_indicadores

    ! --------------------------------------------------
    ! imprimir_tabla_terminal
    ! --------------------------------------------------
    subroutine imprimir_tabla_terminal(ingresos, gastos, categorias, &
                                       meses, total_ing, total_gas, &
                                       promedio_ing, promedio_gas, &
                                       variacion, balance)
        real, intent(in) :: ingresos(:,:), gastos(:,:)
        character(len=*), intent(in) :: categorias(:)
        character(len=*), intent(in) :: meses(:)
        real, intent(in) :: total_ing, total_gas
        real, intent(in) :: promedio_ing, promedio_gas
        real, intent(in) :: variacion(:), balance(:)
        integer :: ncats, nmeses, i, j
        character(len=200) :: fmt_row, fmt_header, fmt_sep
        character(len=30) :: buffer
        real :: max_val
        integer :: ancho_val, ancho_cat, ancho_mes

        ncats = size(ingresos, dim=1)
        nmeses = size(ingresos, dim=2)

        ! Calcular ancho de columna de valores
        max_val = max(maxval(ingresos), maxval(gastos), maxval(abs(balance)), &
                      maxval(abs(variacion)))
        write(buffer, '(F12.2)') max_val
        ancho_val = max(len_trim(adjustl(buffer)) + 2, 10)

        ancho_cat = 15
        ancho_mes = 10

        ! Construir formatos dinamicamente
        write(fmt_header, '(A, I2, A, I2, A)') &
            '(A', ancho_cat, ', A', ancho_mes, ')'

        write(fmt_sep, '(A, I2, A)') &
            '(A, A, ', ancho_cat + ancho_mes + 1, ', A)'

        ! Cabecera
        print *, ""
        print *, "=============================================="
        print *, "  INFORME FINANCIERO MENSUAL"
        print *, "=============================================="
        print *, ""

        ! Tabla de ingresos por categoria
        print *, "--- INGRESOS POR CATEGORIA (USD) ---"
        print fmt_header, "Categoria", "Ene"
        do i = 1, ncats
            write(buffer, '(A)') trim(categorias(i))
            print '(A15, 12F10.2)', adjustl(buffer), (ingresos(i, j), j=1, nmeses)
        end do

        ! Tabla de gastos por categoria
        print *, ""
        print *, "--- GASTOS POR CATEGORIA (USD) ---"
        print fmt_header, "Categoria", "Ene"
        do i = 1, ncats
            write(buffer, '(A)') trim(categorias(i))
            print '(A15, 12F10.2)', adjustl(buffer), (gastos(i, j), j=1, nmeses)
        end do

        ! Variacion mensual
        print *, ""
        print *, "--- VARIACION PORCENTUAL MENSUAL ---"
        print '(A15, 12F10.2)', "Variacion %", (variacion(j), j=1, nmeses)

        ! Balance mensual
        print *, ""
        print *, "--- BALANCE MENSUAL (USD) ---"
        print '(A15, 12F10.2)', "Balance", (balance(j), j=1, nmeses)

        ! Resumen
        print *, ""
        print *, "=============================================="
        print *, "  RESUMEN ANUAL"
        print *, "=============================================="
        print '(A, F12.2)', "Total Ingresos:     ", total_ing
        print '(A, F12.2)', "Total Gastos:       ", total_gas
        print '(A, F12.2)', "Resultado Neto:     ", total_ing - total_gas
        print '(A, F12.2)', "Prom. Ingreso/mes:  ", promedio_ing
        print '(A, F12.2)', "Prom. Gasto/mes:    ", promedio_gas
        print '(EN,A, F12.2)', "Eficiencia (EN):   ", &
              total_ing / max(total_gas, 1.0)
        print '(ES,A, F12.2)', "Eficiencia (ES):   ", &
              total_ing / max(total_gas, 1.0)
        print *, ""

    end subroutine imprimir_tabla_terminal

    ! --------------------------------------------------
    ! exportar_csv
    ! --------------------------------------------------
    subroutine exportar_csv(ingresos, gastos, categorias, meses, &
                            total_ing, total_gas, balance)
        real, intent(in) :: ingresos(:,:), gastos(:,:)
        character(len=*), intent(in) :: categorias(:)
        character(len=*), intent(in) :: meses(:)
        real, intent(in) :: total_ing, total_gas
        real, intent(in) :: balance(:)
        integer :: ncats, nmeses, i, j

        ncats = size(ingresos, dim=1)
        nmeses = size(ingresos, dim=2)

        open(unit=10, file="informe_financiero.csv", status="replace", &
             action="write")

        ! Cabecera
        write(10, '(A)', advance="no") "Categoria"
        do i = 1, nmeses
            write(10, '(A)', advance="no") "," // trim(meses(i))
        end do
        write(10, '(A)') ""

        ! Datos de ingresos
        do i = 1, ncats
            write(10, '(A)', advance="no") &
                "Ingreso_" // trim(adjustl(categorias(i)))
            do j = 1, nmeses
                write(10, '(A, F10.2)', advance="no") ",", ingresos(i, j)
            end do
            write(10, '(A)') ""
        end do

        ! Datos de gastos
        do i = 1, ncats
            write(10, '(A)', advance="no") &
                "Gasto_" // trim(adjustl(categorias(i)))
            do j = 1, nmeses
                write(10, '(A, F10.2)', advance="no") ",", gastos(i, j)
            end do
            write(10, '(A)') ""
        end do

        ! Balance
        write(10, '(A)', advance="no") "Balance"
        do j = 1, nmeses
            write(10, '(A, F10.2)', advance="no") ",", balance(j)
        end do
        write(10, '(A)') ""

        ! Totales
        write(10, '(A, F10.2, A, F10.2)') &
            "Total_Ingresos,", total_ing, ",Total_Gastos,", total_gas

        close(10)
        print *, "CSV exportado: informe_financiero.csv"

    end subroutine exportar_csv

    ! --------------------------------------------------
    ! exportar_latex
    ! --------------------------------------------------
    subroutine exportar_latex(ingresos, gastos, categorias, meses, &
                              total_ing, total_gas, promedio_ing, &
                              promedio_gas, balance)
        real, intent(in) :: ingresos(:,:), gastos(:,:)
        character(len=*), intent(in) :: categorias(:)
        character(len=*), intent(in) :: meses(:)
        real, intent(in) :: total_ing, total_gas
        real, intent(in) :: promedio_ing, promedio_gas
        real, intent(in) :: balance(:)
        integer :: ncats, nmeses, i, j

        ncats = size(ingresos, dim=1)
        nmeses = size(ingresos, dim=2)

        open(unit=20, file="informe_financiero.tex", status="replace", &
             action="write")

        ! Preamble
        write(20, '(A)') "\documentclass[12pt]{article}"
        write(20, '(A)') "\usepackage[utf8]{inputenc}"
        write(20, '(A)') "\usepackage[spanish]{babel}"
        write(20, '(A)') "\usepackage{geometry}"
        write(20, '(A)') "\usepackage{booktabs}"
        write(20, '(A)') "\geometry{a4paper, margin=2.5cm}"
        write(20, '(A)') "\title{Informe Financiero Mensual}"
        write(20, '(A)') "\author{Generado automaticamente desde Fortran}"
        write(20, '(A)') "\date{\today}"
        write(20, '(A)') "\begin{document}"
        write(20, '(A)') "\maketitle"
        write(20, '(A)') ""

        ! Tabla de ingresos
        write(20, '(A)') "\section*{Ingresos por Categoria}"
        write(20, '(A)') "\begin{table}[h]"
        write(20, '(A)') "\centering"
        write(20, '(A)') "\begin{tabular}{l" // repeat("r", nmeses) // "}"
        write(20, '(A)') "\toprule"
        write(20, '(A)', advance="no") "Categoria"
        do i = 1, nmeses
            write(20, '(A)', advance="no") " & " // trim(meses(i))
        end do
        write(20, '(A)') " \\"
        write(20, '(A)') "\midrule"
        do i = 1, ncats
            write(20, '(A)', advance="no") trim(adjustl(categorias(i)))
            do j = 1, nmeses
                write(20, '(A, F10.2)', advance="no") " & $", ingresos(i, j)
            end do
            write(20, '(A)') " \\"
        end do
        write(20, '(A)') "\bottomrule"
        write(20, '(A)') "\end{tabular}"
        write(20, '(A)') "\caption{Ingresos mensuales por categoria (USD).}"
        write(20, '(A)') "\end{table}"
        write(20, '(A)') ""

        ! Tabla de gastos
        write(20, '(A)') "\section*{Gastos por Categoria}"
        write(20, '(A)') "\begin{table}[h]"
        write(20, '(A)') "\centering"
        write(20, '(A)') "\begin{tabular}{l" // repeat("r", nmeses) // "}"
        write(20, '(A)') "\toprule"
        write(20, '(A)', advance="no") "Categoria"
        do i = 1, nmeses
            write(20, '(A)', advance="no") " & " // trim(meses(i))
        end do
        write(20, '(A)') " \\"
        write(20, '(A)') "\midrule"
        do i = 1, ncats
            write(20, '(A)', advance="no") trim(adjustl(categorias(i)))
            do j = 1, nmeses
                write(20, '(A, F10.2)', advance="no") " & $", gastos(i, j)
            end do
            write(20, '(A)') " \\"
        end do
        write(20, '(A)') "\bottomrule"
        write(20, '(A)') "\end{tabular}"
        write(20, '(A)') "\caption{Gastos mensuales por categoria (USD).}"
        write(20, '(A)') "\end{table}"
        write(20, '(A)') ""

        ! Tabla de balance
        write(20, '(A)') "\section*{Balance Mensual}"
        write(20, '(A)') "\begin{table}[h]"
        write(20, '(A)') "\centering"
        write(20, '(A)') "\begin{tabular}{l" // repeat("r", nmeses) // "}"
        write(20, '(A)') "\toprule"
        write(20, '(A)', advance="no") "Mes"
        do i = 1, nmeses
            write(20, '(A)', advance="no") " & " // trim(meses(i))
        end do
        write(20, '(A)') " \\"
        write(20, '(A)') "\midrule"
        write(20, '(A)', advance="no") "Balance"
        do j = 1, nmeses
            write(20, '(A, F10.2)', advance="no") " & $", balance(j)
        end do
        write(20, '(A)') " \\"
        write(20, '(A)') "\bottomrule"
        write(20, '(A)') "\end{tabular}"
        write(20, '(A)') "\caption{Balance mensual (USD).}"
        write(20, '(A)') "\end{table}"
        write(20, '(A)') ""

        ! Resumen
        write(20, '(A)') "\section*{Resumen Anual}"
        write(20, '(A)') "\begin{itemize}"
        write(20, '(A, F10.2, A)') "  \item Total Ingresos: \$", total_ing, ""
        write(20, '(A, F10.2, A)') "  \item Total Gastos: \$", total_gas, ""
        write(20, '(A, F10.2, A)') "  \item Resultado Neto: \$", &
                                    total_ing - total_gas, ""
        write(20, '(A, F10.2, A)') "  \item Promedio Ingreso Mensual: \$", &
                                    promedio_ing, ""
        write(20, '(A, F10.2, A)') "  \item Promedio Gasto Mensual: \$", &
                                    promedio_gas, ""
        write(20, '(A)') "\end{itemize}"

        write(20, '(A)') ""
        write(20, '(A)') "\end{document}"
        close(20)
        print *, "LaTeX exportado: informe_financiero.tex"

    end subroutine exportar_latex

    ! --------------------------------------------------
    ! generar_script_gnuplot
    ! --------------------------------------------------
    subroutine generar_script_gnuplot(ingresos, gastos, categorias, meses, &
                                      total_ing, total_gas)
        real, intent(in) :: ingresos(:,:), gastos(:,:)
        character(len=*), intent(in) :: categorias(:)
        character(len=*), intent(in) :: meses(:)
        real, intent(in) :: total_ing, total_gas
        integer :: ncats, nmeses, i, j

        ncats = size(ingresos, dim=1)
        nmeses = size(ingresos, dim=2)

        ! Generar datos apilados para Gnuplot
        open(unit=30, file="datos_apilados.txt", status="replace", &
             action="write")
        do j = 1, nmeses
            write(30, '(A)', advance="no") trim(meses(j))
            do i = 1, ncats
                write(30, '(A, F12.2)', advance="no") " ", ingresos(i, j)
            end do
            write(30, '(A)') ""
        end do
        close(30)

        ! Generar datos de gastos
        open(unit=31, file="datos_gastos.txt", status="replace", &
             action="write")
        do j = 1, nmeses
            write(31, '(A)', advance="no") trim(meses(j))
            do i = 1, ncats
                write(31, '(A, F12.2)', advance="no") " ", gastos(i, j)
            end do
            write(31, '(A)') ""
        end do
        close(31)

        ! Generar script .gp
        open(unit=32, file="informe_financiero.gp", status="replace", &
             action="write")
        write(32, '(A)') "set terminal pdfcairo enhanced font 'Helvetica,12'"
        write(32, '(A)') "set output 'informe_financiero.pdf'"
        write(32, '(A)') "set title 'Ingresos y Gastos Mensuales'"
        write(32, '(A)') "set xlabel 'Mes'"
        write(32, '(A)') "set ylabel 'Monto (USD)'"
        write(32, '(A)') "set grid ytics"
        write(32, '(A)') "set style data histograms"
        write(32, '(A)') "set style histogram rowstacked"
        write(32, '(A)') "set style fill solid 0.8 border -1"
        write(32, '(A)') "set boxwidth 0.9"
        write(32, '(A, I1, A)') "set key outside above"

        ! Comando plot con todas las columnas
        write(32, '(A)', advance="no") "plot "
        do i = 1, ncats
            if (i > 1) write(32, '(A)', advance="no") ", "
            write(32, '(A, I1, A, A, A)', advance="no") &
                "'datos_apilados.txt' using ", i + 1, &
                " title 'Ingreso ", trim(categorias(i)), "'"
        end do
        do i = 1, ncats
            write(32, '(A)', advance="no") ", "
            write(32, '(A, I1, A, A, A)', advance="no") &
                "'datos_gastos.txt' using ", i + 1, &
                " title 'Gasto ", trim(categorias(i)), "'"
        end do
        write(32, '(A)') ""
        close(32)
        print *, "Gnuplot script: informe_financiero.gp"

    end subroutine generar_script_gnuplot

end module report_utils
```

Ahora, el programa principal que usa el módulo:

```fortran
! ===================================================
! generar_informe.f90 - Programa principal
! ===================================================
program generar_informe
    use report_utils
    implicit none

    ! Dimensiones: 3 categorias, 12 meses
    integer, parameter :: ncats = 3
    integer, parameter :: nmeses = 12

    character(len=10) :: categorias(ncats)
    character(len=5)  :: meses(nmeses)

    real :: ingresos(ncats, nmeses)
    real :: gastos(ncats, nmeses)

    real :: total_ing, total_gas
    real :: promedio_ing, promedio_gas
    real :: variacion(nmeses)
    real :: balance(nmeses)

    integer :: i, j, exit_status
    character(len=200) :: comando

    ! Nombres de categorias y meses
    categorias = ["Servicios", "Productos", "Consultoría"]
    meses = ["Ene", "Feb", "Mar", "Abr", "May", "Jun", &
             "Jul", "Ago", "Sep", "Oct", "Nov", "Dic"]

    ! Datos sinteticos de ingresos (categoria x mes)
    ! Servicios: crecimiento gradual
    ingresos(1, :) = [5000.0, 5200.0, 5100.0, 5500.0, 5800.0, 6000.0, &
                      6200.0, 6100.0, 6500.0, 7000.0, 7200.0, 7500.0]
    ! Productos: estacional (picos en marzo y diciembre)
    ingresos(2, :) = [3000.0, 2800.0, 4500.0, 3200.0, 3100.0, 3500.0, &
                      3300.0, 3400.0, 3600.0, 3800.0, 4200.0, 5000.0]
    ! Consultoria: estable
    ingresos(3, :) = [2000.0, 2100.0, 2050.0, 2200.0, 2150.0, 2300.0, &
                      2250.0, 2400.0, 2350.0, 2500.0, 2450.0, 2600.0]

    ! Datos sinteticos de gastos
    ! Servicios: operativos
    gastos(1, :) = [3500.0, 3450.0, 3600.0, 3550.0, 3700.0, 3800.0, &
                    3750.0, 3900.0, 3850.0, 4000.0, 3950.0, 4100.0]
    ! Productos: compras
    gastos(2, :) = [2000.0, 1900.0, 2800.0, 2100.0, 2000.0, 2200.0, &
                    2100.0, 2150.0, 2300.0, 2400.0, 2600.0, 3200.0]
    ! Consultoria: overhead
    gastos(3, :) = [800.0,  820.0,  810.0,  850.0,  830.0,  880.0, &
                    860.0,  900.0,  890.0,  920.0,  910.0,  950.0]

    ! Paso 1: Calcular indicadores
    print *, "=============================================="
    print *, "  GENERADOR DE INFORMES FINANCIEROS"
    print *, "=============================================="
    print *, "Procesando datos financieros..."
    print *, ""

    call calcular_indicadores(ingresos, gastos, total_ing, total_gas, &
                              promedio_ing, promedio_gas, variacion, &
                              balance, categorias)

    ! Paso 2: Imprimir tabla en terminal
    call imprimir_tabla_terminal(ingresos, gastos, categorias, meses, &
                                 total_ing, total_gas, promedio_ing, &
                                 promedio_gas, variacion, balance)

    ! Paso 3: Exportar a CSV
    call exportar_csv(ingresos, gastos, categorias, meses, &
                      total_ing, total_gas, balance)

    ! Paso 4: Exportar a LaTeX
    call exportar_latex(ingresos, gastos, categorias, meses, &
                        total_ing, total_gas, promedio_ing, &
                        promedio_gas, balance)

    ! Paso 5: Generar script de Gnuplot
    call generar_script_gnuplot(ingresos, gastos, categorias, meses, &
                                total_ing, total_gas)

    ! Paso 6: Ejecutar Gnuplot si esta disponible
    print *, ""
    print *, "--- EJECUTANDO GNUPLOT ---"
    comando = "gnuplot informe_financiero.gp 2>&1"
    call execute_command_line(trim(comando), exitstat=exit_status)

    if (exit_status == 0) then
        print *, "Grafico generado: informe_financiero.pdf"
    else
        print *, "Gnuplot no disponible o error (codigo:", exit_status, ")"
        print *, "Puedes ejecutar manualmente: gnuplot informe_financiero.gp"
    end if

    print *, ""
    print *, "=============================================="
    print *, "  INFORME GENERADO EXITOSAMENTE"
    print *, "=============================================="
    print *, "Archivos generados:"
    print *, "  - informe_financiero.csv"
    print *, "  - informe_financiero.tex"
    print *, "  - informe_financiero.gp"
    print *, "  - datos_apilados.txt"
    print *, "  - datos_gastos.txt"
    if (exit_status == 0) then
        print *, "  - informe_financiero.pdf"
    end if

end program generar_informe
```

**Análisis línea por línea**:

**Módulo `report_utils`**:

Líneas 1-8: `module report_utils` con `private` y `public`. Esto encapsula el módulo: solo las subrutinas listadas en `public` son accesibles desde fuera. El resto (variables internas, funciones auxiliares) quedan ocultas.

Líneas 9-10: Constantes públicas `MAX_CATEGORIAS` y `MAX_MESES`. Cualquier programa que use el módulo puede referenciarlas como `report_utils%MAX_CATEGORIAS` o directamente si se usa `use report_utils`.

Subrutina `calcular_indicadores`:

Líneas 17-24: Argumentos de la subrutina. Los arrays `ingresos(:,:)` y `gastos(:,:)` son de forma asumida (assumed shape), lo que permite pasar arreglos de cualquier tamaño. Los argumentos `intent(in)` son de solo lectura, `intent(out)` son de solo escritura.

Líneas 25-26: `size(ingresos, dim=1)` devuelve el número de categorías (filas) y `dim=2` el número de meses (columnas). Esta es la forma moderna y flexible de determinar dimensiones.

Líneas 28-31: Cálculos de totales y promedios. `sum` suma todos los elementos de un arreglo, sin importar sus dimensiones.

Líneas 33-40: Cálculo de variación porcentual mes a mes. Para el primer mes la variación es 0. Para los siguientes, se calcula como `(mes_actual - mes_anterior) / mes_anterior * 100`. Nota el uso de `sum(ingresos(:, i))` para sumar todas las categorías de un mes.

Líneas 42-46: Cálculo del balance mensual: ingresos totales del mes menos gastos totales del mes.

Subrutina `imprimir_tabla_terminal`:

Líneas 52-61: Argumentos. Acepta los arrays de datos, las etiquetas, y los indicadores calculados.

Líneas 68-71: Cálculo del ancho de columna óptimo. `maxval` encuentra el máximo valor en los arreglos de ingresos, gastos, balance y variación. Se convierte a string y se mide su longitud.

Línea 72: `ancho_val = max(len_trim(adjustl(buffer)) + 2, 10)` — Ancho mínimo de 10 caracteres, o el necesario para el valor más grande más 2 de margen.

Líneas 74-80: Se construyen formatos dinámicos. `fmt_header` crea un formato como `(A15, A10)` para la cabecera. En una implementación completa, esto se usaría con los 12 meses.

Líneas 82-85: Línea separadora de asteriscos. `repeat("*", N)` genera N asteriscos.

Líneas 87-120: Imprimen las tablas de ingresos, gastos, variación y balance. Usa formato `(A15, 12F10.2)` — 15 caracteres para el nombre y 12 valores de 10 caracteres cada uno.

Líneas 122-136: Resumen anual con formato amigable. Incluye ejemplos de `EN` y `ES` para mostrar la diferencia (aunque el resultado numérico es similar, el formato cambia).

Subrutina `exportar_csv`:

Líneas 140-143: Argumentos.

Líneas 148-150: `open` con `status="replace"` sobreescribe el archivo CSV.

Líneas 152-156: Cabecera del CSV con nombres de meses separados por comas. `advance="no"` evita el salto de línea hasta tener la fila completa.

Líneas 158-166: Datos de ingresos: cada fila comienza con "Ingreso_NombreCategoria" seguido de 12 valores.

Líneas 168-176: Datos de gastos: mismo patrón.

Líneas 178-183: Fila de balance mensual.

Líneas 185-188: Fila de totales.

Subrutina `exportar_latex`:

Líneas 192-197: Argumentos.

Líneas 201-212: Preámbulo LaTeX estándar. Incluye `booktabs` para líneas de tabla profesionales (`\toprule`, `\midrule`, `\bottomrule`).

Líneas 214-237: Tabla de ingresos con formato LaTeX. Usa `repeat("r", nmeses)` para generar las columnas derechas dinámicamente.

Líneas 239-262: Tabla de gastos (misma estructura).

Líneas 264-285: Tabla de balance mensual.

Líneas 287-301: Resumen anual con itemize.

Subrutina `generar_script_gnuplot`:

Líneas 305-309: Argumentos.

Líneas 312-321: Genera `datos_apilados.txt` con formato: mes valor1 valor2 valor3. Una fila por mes.

Líneas 323-332: Genera `datos_gastos.txt` con el mismo formato.

Líneas 334-356: Genera el script `.gp`. La línea más compleja es el `plot` que combina todas las series. Se construye dinámicamente concatenando los nombres de las categorías.

Líneas 358-361: Cierra y reporta.

**Programa principal**:

Línea 2: `use report_utils` — Importa el módulo. Todas las subrutinas públicas están disponibles.

Líneas 7-8: Define las dimensiones fijas del problema: 3 categorías, 12 meses.

Líneas 10-11: Declara los arreglos de etiquetas.

Líneas 13-14: Declara los arreglos de datos: `ingresos(ncats, nmeses)` y `gastos(ncats, nmeses)`.

Líneas 16-19: Declara variables para los indicadores calculados.

Líneas 23-24: Asigna nombres a las categorías y meses.

Líneas 26-40: Asigna datos sintéticos. Los ingresos de servicios crecen gradualmente, los de productos son estacionales con picos en marzo y diciembre, los de consultoría son estables. Los gastos siguen patrones similares.

Líneas 42-49: Mensaje inicial.

Líneas 51-53: Llama a `calcular_indicadores`.

Línea 56: Llama a `imprimir_tabla_terminal`.

Línea 59: Llama a `exportar_csv`.

Línea 62: Llama a `exportar_latex`.

Línea 65: Llama a `generar_script_gnuplot`.

Líneas 68-77: Ejecuta Gnuplot con `execute_command_line`. Verifica el código de salida.

Líneas 79-91: Mensaje final con lista de archivos generados.

**Salida esperada** (terminal):

```
==============================================
  GENERADOR DE INFORMES FINANCIEROS
==============================================
Procesando datos financieros...

==============================================
  INFORME FINANCIERO MENSUAL
==============================================

--- INGRESOS POR CATEGORIA (USD) ---
Categoria             Ene       Feb       Mar       Abr       May       Jun       Jul       Ago       Sep       Oct       Nov       Dic
Servicios         5000.00   5200.00   5100.00   5500.00   5800.00   6000.00   6200.00   6100.00   6500.00   7000.00   7200.00   7500.00
Productos         3000.00   2800.00   4500.00   3200.00   3100.00   3500.00   3300.00   3400.00   3600.00   3800.00   4200.00   5000.00
Consultoría       2000.00   2100.00   2050.00   2200.00   2150.00   2300.00   2250.00   2400.00   2350.00   2500.00   2450.00   2600.00

--- GASTOS POR CATEGORIA (USD) ---
Categoria             Ene       Feb       Mar       Abr       May       Jun       Jul       Ago       Sep       Oct       Nov       Dic
Servicios         3500.00   3450.00   3600.00   3550.00   3700.00   3800.00   3750.00   3900.00   3850.00   4000.00   3950.00   4100.00
Productos         2000.00   1900.00   2800.00   2100.00   2000.00   2200.00   2100.00   2150.00   2300.00   2400.00   2600.00   3200.00
Consultoría        800.00    820.00    810.00    850.00    830.00    880.00    860.00    900.00    890.00    920.00    910.00    950.00

--- VARIACION PORCENTUAL MENSUAL ---
Variacion %          0.00      5.71     -4.55      8.70      4.76      4.26      3.33     -1.52      5.88      6.25      3.28      4.59

--- BALANCE MENSUAL (USD) ---
Balance           3700.00   3930.00   3240.00   4300.00   4420.00   4620.00   4740.00   4550.00   5010.00   5480.00   5490.00   5350.00

==============================================
  RESUMEN ANUAL
==============================================
Total Ingresos:     130650.00
Total Gastos:        89770.00
Resultado Neto:      40880.00
Prom. Ingreso/mes:    3629.17
Prom. Gasto/mes:      2493.61
Eficiencia (EN):         1.46
Eficiencia (ES):         1.46

--- EJECUTANDO GNUPLOT ---
 Gnuplot no disponible o error (codigo: 127)
 Puedes ejecutar manualmente: gnuplot informe_financiero.gp

==============================================
  INFORME GENERADO EXITOSAMENTE
==============================================
Archivos generados:
  - informe_financiero.csv
  - informe_financiero.tex
  - informe_financiero.gp
  - datos_apilados.txt
  - datos_gastos.txt
```

**Archivos generados**:

`informe_financiero.csv`:

```
Categoria,Ene,Feb,Mar,Abr,May,Jun,Jul,Ago,Sep,Oct,Nov,Dic
Ingreso_Servicios,  5000.00,  5200.00,  5100.00,  5500.00,  5800.00,  6000.00,  6200.00,  6100.00,  6500.00,  7000.00,  7200.00,  7500.00
Ingreso_Productos,  3000.00,  2800.00,  4500.00,  3200.00,  3100.00,  3500.00,  3300.00,  3400.00,  3600.00,  3800.00,  4200.00,  5000.00
Ingreso_Consultoria,  2000.00,  2100.00,  2050.00,  2200.00,  2150.00,  2300.00,  2250.00,  2400.00,  2350.00,  2500.00,  2450.00,  2600.00
Gasto_Servicios,  3500.00,  3450.00,  3600.00,  3550.00,  3700.00,  3800.00,  3750.00,  3900.00,  3850.00,  4000.00,  3950.00,  4100.00
Gasto_Productos,  2000.00,  1900.00,  2800.00,  2100.00,  2000.00,  2200.00,  2100.00,  2150.00,  2300.00,  2400.00,  2600.00,  3200.00
Gasto_Consultoria,  800.00,  820.00,  810.00,  850.00,  830.00,  880.00,  860.00,  900.00,  890.00,  920.00,  910.00,  950.00
Balance,  3700.00,  3930.00,  3240.00,  4300.00,  4420.00,  4620.00,  4740.00,  4550.00,  5010.00,  5480.00,  5490.00,  5350.00
Total_Ingresos, 130650.00,Total_Gastos, 89770.00
```

`informe_financiero.gp`:

```gnuplot
set terminal pdfcairo enhanced font 'Helvetica,12'
set output 'informe_financiero.pdf'
set title 'Ingresos y Gastos Mensuales'
set xlabel 'Mes'
set ylabel 'Monto (USD)'
set grid ytics
set style data histograms
set style histogram rowstacked
set style fill solid 0.8 border -1
set boxwidth 0.9
set key outside above
plot 'datos_apilados.txt' using 2 title 'Ingreso Servicios', \
     'datos_apilados.txt' using 3 title 'Ingreso Productos', \
     'datos_apilados.txt' using 4 title 'Ingreso Consultoría', \
     'datos_gastos.txt' using 2 title 'Gasto Servicios', \
     'datos_gastos.txt' using 3 title 'Gasto Productos', \
     'datos_gastos.txt' using 4 title 'Gasto Consultoría'
```

**Compilación y ejecución**:

Para compilar los dos archivos:

```bash
gfortran -std=f2018 -c report_utils.f90 -o report_utils.o
gfortran -std=f2018 generar_informe.f90 report_utils.o -o generar_informe
./generar_informe
```

O en un solo paso:

```bash
gfortran -std=f2018 report_utils.f90 generar_informe.f90 -o generar_informe
./generar_informe
```

**Análisis de errores típicos del mini-proyecto**:

```fortran
! Error: no compilar el modulo antes del programa principal
gfortran -std=f2018 generar_informe.f90 -o generar_informe
```

Mensaje de gfortran: `Fatal Error: Cannot open module file 'report_utils.mod' for reading at (1): No such file or directory`. Solución: compila el módulo primero (`gfortran -c report_utils.f90`) o compila ambos archivos en un solo comando.

```fortran
! Error: usar intent(out) en un arreglo y no asignar todos los elementos
subroutine prueba(x)
    real, intent(out) :: x(:)
    ! x(1) = 1.0  ! Si solo asignamos el primero...
end subroutine
```

Mensaje: no hay error del compilador, pero los elementos no asignados quedan indefinidos. Solución: inicializa todo el arreglo con `x = 0.0` al inicio de la subrutina.

```fortran
! Error: confusion entre argumentos posicionales y con palabra clave
call exportar_csv(ingresos, gastos, cat, meses, total_ing, total_gas, balance)
! El orden de los argumentos debe coincidir exactamente con la definicion
```

Mensaje de gfortran: `Error: Type mismatch in argument 'categorias' at (1); passed REAL(4) to CHARACTER(*)`. Solución: verifica que el orden y tipo de cada argumento coincida con la interfaz. En Fortran moderno puedes usar argumentos con palabra clave: `call exportar_csv(ingresos=ingresos, gastos=gastos, categorias=cat, ...)`.

---

Con este mini-proyecto has construido un pipeline completo de generación de reportes financieros. Has utilizado:

- **Formatos avanzados**: `F10.2`, `EN`, `ES`, `A` con ancho, `I`, `X`, `/`, `TC`.
- **Tablas dinámicas**: ancho de columna calculado en tiempo de ejecución.
- **Exportación a CSV**: con formato controlado.
- **Exportación a LaTeX**: documento `.tex` completamente funcional.
- **Integración con Gnuplot**: script `.gp` generado y ejecutado.
- **Cadenas avanzadas**: `trim`, `adjustl`, `len_trim`, `index`, `repeat`, concatenación.
- **Módulos**: encapsulación con `public`/`private`.
- **Arreglos de forma asumida**: subrutinas que aceptan cualquier tamaño.
- **Llamada al sistema**: `execute_command_line` con verificación de estado.

Este patrón es directamente aplicable a cualquier dominio que requiera generación de reportes: simulaciones científicas, análisis financiero, monitoreo de sistemas, procesamiento de datos experimentales, y más. Lo que cambia son los datos y los cálculos; la arquitectura del pipeline permanece.

En el próximo capítulo exploraremos la lectura y escritura de archivos binarios, un tema fundamental para el procesamiento eficiente de grandes volúmenes de datos en aplicaciones HPC.