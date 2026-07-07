# Capítulo 0: Fundamentos de Fortran Moderno

> Un recorrido completo desde la instalación del compilador hasta la construcción de una calculadora estadística interactiva, explorando cada concepto con explicaciones conceptuales profundas, analogías del mundo real, código funcional comentado en español, análisis línea por línea, errores típicos y mini-proyectos prácticos.

---

**Stack tecnológico**: GNU/Linux (Ubuntu 22.04+), gfortran 13+, Estándares Fortran 95/2003/2008/2018. Todo el código del capítulo se compila con:

```bash
gfortran -std=f2018 -Wall -Wextra -o programa archivo.f90
```

---

## 1. Introducción e instalación

Fortran no es un lenguaje muerto. Es, de hecho, uno de los lenguajes de programación más longevos y aún hoy sigue siendo insustituible en computación científica, simulación numérica, modelado climático, dinámica de fluidos computacional (CFD) y análisis por elementos finitos. Nacido en la década de 1950 como el primer lenguaje de alto nivel de la historia, ha evolucionado dramáticamente: Fortran 77 trajo la estructuración básica, Fortran 90/95 introdujo módulos, tipos derivados y arrays con operaciones vectoriales, Fortran 2003 añadió programación orientada a objetos e interoperabilidad con C, Fortran 2008 incorporó concurrencia con coarrays, y Fortran 2018 continuó mejorando la paralelización. El Fortran moderno se parece muy poco a su abuelo de los 70: tiene tipos de datos ricos, control de flujo estructurado, manejo de memoria dinámica y capacidades de paralelismo nativas.

¿Por qué seguir aprendiendo Fortran en pleno siglo XXI? La respuesta es doble. Por un lado, existe una cantidad colosal de código legacy —millones de líneas— en instituciones como la NASA, el ECMWF (Centro Europeo de Previsiones Meteorológicas a Medio Plazo) o los laboratorios nacionales de Estados Unidos, que sigue en producción y necesita ser mantenido, extendido y modernizado. Por otro lado, Fortran sigue siendo el rey indiscutible de la computación numérica de alto rendimiento: su semántica de arrays permite al compilador hacer optimizaciones agresivas que lenguajes como Python o incluso C++ no pueden igualar sin bibliotecas externas. Cuando necesitas exprimir cada ciclo de reloj en una simulación que correrá durante semanas en una supercomputadora, Fortran sigue siendo la herramienta más natural.

Pensemos en Fortran como una caja de herramientas de alta precisión. No es un martillo multiusos como Python, ni un destornillador de precisión como C. Es una fresadora CNC: está diseñada para un tipo de tarea muy específico (la computación numérica intensiva) y en ese dominio no tiene rival. Aprender Fortran moderno no es aprender un lenguaje antiguo; es aprender las mejores prácticas de un lenguaje que ha ido incorporando selectivamente lo mejor de la ingeniería de software moderna sin perder su esencia: la eficiencia numérica.

El ecosistema actual de Fortran está dominado por dos compiladores principales: gfortran (parte de GCC, gratuito y de código abierto) e ifort/ifx de Intel. Nosotros usaremos gfortran por su disponibilidad universal, su buen soporte de los estándares modernos y su integración con herramientas de desarrollo GNU. Gfortran soporta completamente Fortran 95 y la mayor parte de Fortran 2003, 2008 y 2018, lo que lo convierte en una plataforma excelente tanto para aprender como para producir código de calidad.

Antes de escribir nuestra primera línea de código, es importante entender cómo piensa Fortran. A diferencia de C o Python, donde todo es expresión y función, Fortran está diseñado alrededor del concepto de "programa principal" que llama a subprogramas. La unidad mínima ejecutable es un `program` que termina con `end program`. No hay una función `main()` explícita como en C; el punto de entrada es el bloque `program` mismo. Esta estructura refleja los orígenes de Fortran como un lenguaje para transformar fórmulas matemáticas en código de manera directa.

Una característica fundamental que distingue a Fortran de casi cualquier otro lenguaje moderno es la separación entre el código ejecutable y las declaraciones. En Fortran, todo debe ser declarado antes de ser usado (especialmente si usamos `implicit none`, que debería ser siempre nuestra primera regla). Esta disciplina de declaración explícita no es burocracia innecesaria: es precisamente lo que permite al compilador hacer verificaciones de tipos en tiempo de compilación y generar código máquina óptimo, porque sabe exactamente con qué tipo de datos está trabajando en cada momento.

La instalación de gfortran en Ubuntu es directa, pero vale la pena entender qué estamos instalando. El paquete `gfortran` es parte del conjunto de compiladores GNU (GCC). Al instalarlo, obtenemos no solo el compilador sino también las bibliotecas de runtime de Fortran, que incluyen funciones intrínsecas matemáticas, de E/S y de manejo de cadenas. Estas bibliotecas son el equivalente a la biblioteca estándar de C (`libc`), pero especializadas para el modelo de computación de Fortran.

Finalmente, una reflexión sobre la compilación. En lenguajes interpretados como Python, cada ejecución paga el costo de interpretación. En lenguajes compilados como Fortran, pagamos una vez el costo de traducción a código máquina y luego la ejecución es prácticamente tan rápida como el hardware lo permite. La línea `gfortran -std=f2018 -Wall -Wextra -o programa archivo.f90` hace varias cosas: activa el estándar 2018 (asegurando compatibilidad y rechazando extensiones obsoletas), habilita todas las advertencias (`-Wall -Wextra`) para detectar errores sutiles tempranamente, y produce un ejecutable con el nombre que especifiquemos. Esta disciplina de compilación estricta desde el principio ahorrará incontables horas de depuración.

### Instalación de gfortran

```bash
# Actualizar la lista de paquetes
sudo apt update

# Instalar gfortran (versión 13+ en Ubuntu 24.04)
sudo apt install gfortran

# Verificar la instalación
gfortran --version
```

### Hola mundo en Fortran Moderno

```fortran
! programa/hola_mundo.f90
program hola_mundo
    implicit none                          ! Obliga a declarar todas las variables

    ! La instrucción write con asterisco escribe en la salida estándar
    write(*, *) '¡Hola, mundo desde Fortran Moderno!'

end program hola_mundo
```

**Análisis línea por línea**:
- `program hola_mundo`: Define el inicio del programa principal con nombre `hola_mundo`. Sin esta línea, el compilador no sabe dónde comienza la unidad de programa. Si faltara, gfortran asumiría un archivo con sentencias sueltas, pero en un programa completo es obligatorio.
- `implicit none`: Desactiva la declaración implícita de tipos. En Fortran antiguo, las variables que comenzaban con letras i-n eran enteras y el resto reales. Esta línea fuerza a declarar todo explícitamente, evitando errores por erratas. Si se omite, una variable escrita mal se convierte silenciosamente en una nueva variable, causando bugs devastadores.
- `write(*, *) '¡Hola, mundo desde Fortran Moderno!'`: La instrucción de escritura. El primer `*` indica unidad de salida estándar (la terminal). El segundo `*` indica formato libre (el compilador decide el formato). Si el formato libre no estuviera disponible (estándares anteriores al 95), habría que usar `format` explícito.
- `end program hola_mundo`: Marca el final del programa. Debe coincidir con el nombre del `program`. Si falta, el compilador asume el final del archivo, pero es buena práctica incluirlo porque mejora la legibilidad y el compilador puede verificar que los bloques están correctamente cerrados.

**Salida esperada**:
```
¡Hola, mundo desde Fortran Moderno!
```

**Errores típicos**:

1. **Error: `implicit none` omitido y variable mal escrita**
```fortran
program error_implicit
    ! implicit none   ! Descomentar para evitar el error
    integer :: x
    x = 5
    y = x + 1         ! y no está declarada; implicit none la crea como real
    print *, y
end program error_implicit
```
Mensaje real: No hay error de compilación (porque `implicit` está activo por defecto), pero `y` se declara implícitamente como `real` y el resultado es incorrecto. Con `implicit none`, el compilador daría: `Error: Symbol 'y' at (1) has no IMPLICIT type`.

2. **Error: nombre del programa no coincide en el cierre**
```fortran
program hola
    implicit none
    write(*, *) 'Hola'
end program holaa
```
Mensaje real: `Error: Expected 'hola' at (1) instead of 'holaa' in END PROGRAM statement`. El compilador verifica que el nombre coincida.

**Mini-proyecto: Saludo personalizado con medición de tiempo**

Este programa saluda al usuario, lee su nombre y la cantidad de veces que quiere ser saludado, y mide cuánto tarda la ejecución. Te familiarizarás con entrada estándar, variables, y la función intrínseca `system_clock`.

```fortran
program saludo_personalizado
    implicit none

    ! Declaración de variables
    character(len=50) :: nombre           ! Cadena para almacenar el nombre
    integer :: repeticiones                ! Número de saludos
    integer :: i                            ! Variable de iteración
    integer :: t_inicio, t_final, tasa     ! Variables para medir tiempo

    ! Leer el nombre del usuario
    write(*, *) '¿Cómo te llamas?'
    read(*, *) nombre

    ! Leer el número de veces
    write(*, *) '¿Cuántas veces quieres ser saludado?'
    read(*, *) repeticiones

    ! Tomar tiempo inicial
    call system_clock(t_inicio, tasa)

    ! Bucle de saludos
    do i = 1, repeticiones
        write(*, *) 'Hola, ', trim(nombre), '!'
    end do

    ! Tomar tiempo final y calcular
    call system_clock(t_final)
    write(*, *) '---'
    write(*, *) 'Tiempo total: ', real(t_final - t_inicio) / real(tasa), ' segundos'

end program saludo_personalizado
```

**Análisis línea por línea**:
- `character(len=50) :: nombre`: Declara una cadena de hasta 50 caracteres. En Fortran, las cadenas tienen longitud fija. Si no se usa `len=50`, el valor por defecto es 1, lo que truncaría el nombre.
- `call system_clock(t_inicio, tasa)`: Llama a la subrutina intrínseca que lee el reloj del sistema. `tasa` es el número de "ticks" por segundo.
- `do i = 1, repeticiones`: Inicia un bucle que itera desde 1 hasta `repeticiones`. La variable `i` se incrementa automáticamente. En Fortran, `do` siempre cuenta hacia arriba a menos que se use `step` negativo.
- `trim(nombre)`: Función intrínseca que elimina espacios finales de la cadena. Si se omitiera, el nombre aparecería seguido de espacios hasta llenar los 50 caracteres.
- `real(t_final - t_inicio) / real(tasa)`: Convierte enteros a reales para obtener una división con decimales. Sin `real()`, la división sería entera, truncando el resultado a 0.

**Salida esperada** (interactiva):
```
 ¿Cómo te llamas?
> Carlos
 ¿Cuántas veces quieres ser saludado?
> 3
 Hola, Carlos!
 Hola, Carlos!
 Hola, Carlos!
 ---
 Tiempo total:   1.20000005E-04  segundos
```

---

## 2. Variables y tipos de datos

Imaginemos que estamos organizando un almacén de datos. Necesitamos diferentes tipos de contenedores para diferentes tipos de mercancía: cajas pequeñas para números enteros, grandes tanques para números con decimales, botellas etiquetadas para letras, interruptores de luz para valores de verdadero/falso. Fortran, como todo lenguaje de programación, necesita saber exactamente qué tipo de contenedor estamos usando para cada dato, porque el espacio en memoria y la forma de interpretar los bits depende del tipo. Un número entero ocupa menos espacio que un número real, y un carácter necesita un esquema de codificación completamente distinto.

Fortran ofrece cinco tipos de datos intrínsecos (incorporados en el lenguaje): `integer`, `real`, `complex`, `character` y `logical`. Además, existe `double precision` (doble precisión) que es una forma de `real` con más dígitos de precisión. En Fortran moderno, se recomienda usar `real(kind=8)` o `real(selected_real_kind(15, 307))` en lugar de `double precision`, pero este último sigue siendo ampliamente usado por su simplicidad y legibilidad.

La filosofía de tipado en Fortran es estricta y explícita. A diferencia de Python, donde una variable puede cambiar de tipo en tiempo de ejecución, en Fortran el tipo de cada variable se fija en tiempo de compilación. Esto no es una limitación: es una ventaja de rendimiento. El compilador sabe exactamente cuántos bytes reservar para cada variable y qué instrucciones máquina generar para operar con ella. ¿Qué ocurre si sumamos un entero con un real? Fortran convierte automáticamente el entero a real antes de la operación (conversión implícita), pero el resultado será `real`. Esta flexibilidad tiene un precio: si no somos conscientes de las conversiones, podemos perder precisión silenciosamente.

Los tipos en Fortran tienen un atributo llamado `kind` que determina el tamaño en memoria y la precisión. Un `integer` por defecto suele ser de 4 bytes (rango de -2.147.483.648 a 2.147.483.647), pero podemos especificar `integer(kind=8)` para enteros de 8 bytes. Para `real`, el valor por defecto es de 4 bytes (precisión simple, unos 7 dígitos decimales), y `real(kind=8)` nos da precisión doble (unos 15 dígitos). Esta granularidad permite al programador científico ajustar el equilibrio entre precisión y uso de memoria, algo crítico cuando trabajamos con arrays de millones de elementos.

El tipo `complex` es un ciudadano de primera clase en Fortran, no una biblioteca externa. Un número complejo tiene parte real y parte imaginaria, y Fortran entiende operaciones como multiplicación, división, raíces, etc., directamente. Esto es una ventaja enorme sobre lenguajes como C, donde las operaciones con complejos requieren bibliotecas especiales. Para el científico que trabaja con análisis de Fourier, mecánica cuántica o procesamiento de señales, esta característica hace que el código sea mucho más legible y menos propenso a errores.

El tipo `character` en Fortran merece una mención especial. A diferencia de C, donde una cadena es un array de caracteres terminado en nulo, en Fortran una cadena tiene longitud fija y se pasa completa (no hay terminador nulo). La declaración `character(len=n) :: variable` reserva `n` bytes. Si almacenamos un texto más corto, el resto se rellena con espacios. Esta filosofía de "todo o nada" viene del mundo de las tarjetas perforadas, donde cada tarjeta tenía exactamente 80 columnas. Hoy puede parecer arcaico, pero tiene la ventaja de que no hay desbordamientos de búfer como en C: Fortran simplemente trunca o rellena con espacios.

Finalmente, `logical` es el tipo más simple: solo puede ser `.true.` o `.false.` (nótese los puntos alrededor de los valores, una sintaxis heredada del Fortran original). Los operadores lógicos son `.and.`, `.or.`, `.not.`, `.eqv.` y `.neqv.`. Aunque parezca un tipo menor, el `logical` es la base de todo el control de flujo: sin valores booleanos no podríamos tener `if`, `select case` ni bucles condicionales. Es el interruptor que enciende o apaga el flujo del programa.

### Declaración y uso de todos los tipos básicos

```fortran
! programas/tipos_basicos.f90
program tipos_basicos
    implicit none

    ! integer: números enteros (4 bytes por defecto)
    integer :: edad = 30
    integer(kind=8) :: poblacion_mundial = 8000000000_8

    ! real: números con decimales (precisión simple, ~7 dígitos)
    real :: temperatura = 36.5
    real :: pi_aprox = 3.14159265

    ! double precision: precisión doble (~15 dígitos)
    double precision :: constante_planck = 6.62607015d-34

    ! complex: números complejos
    complex :: z = (1.0, 2.0)
    complex :: w

    ! character: cadenas de texto
    character(len=40) :: nombre = 'Ada Lovelace'

    ! logical: valores booleanos
    logical :: es_cientifica = .true.

    ! Mostrar valores en pantalla
    write(*, *) '=== DEMOSTRACIÓN DE TIPOS BÁSICOS ==='
    write(*, *)
    write(*, *) 'Edad   (integer): ', edad
    write(*, *) 'Población (integer*8): ', poblacion_mundial
    write(*, *) 'Temperatura    (real): ', temperatura
    write(*, *) 'Pi aproximado  (real): ', pi_aprox
    write(*, *) 'Constante Planck (d.p.): ', constante_planck
    write(*, *) 'Número complejo (z): ', z

    ! Operaciones con complejos
    w = z * (0.0, 1.0)   ! Multiplicar por i (rotación 90°)
    write(*, *) 'z * i  (rotación): ', w

    ! Funciones intrínsecas para complejos
    write(*, *) 'Parte real de z:     ', real(z)
    write(*, *) 'Parte imaginaria:    ', aimag(z)
    write(*, *) 'Módulo (magnitud):   ', abs(z)
    write(*, *) 'Argumento (fase):    ', atan2(aimag(z), real(z))

    ! Operaciones con caracteres
    write(*, *) 'Longitud del nombre: ', len_trim(nombre)
    write(*, *) 'Nombre en mayúsculas: ', doble(nombre)  ! doble no es intrínseca pero ilustra el concepto

    ! Uso de lógicos
    if (es_cientifica) then
        write(*, *) nombre, ' es una científica.'
    end if

end program tipos_basicos
```

**Análisis línea por línea**:
- `integer(kind=8) :: poblacion_mundial = 8000000000_8`: Declara un entero de 8 bytes con valor inicial. El sufijo `_8` en el literal indica que la constante es de tipo `integer(kind=8)`. Si se omitiera el sufijo, el compilador podría truncar el valor a 4 bytes, dando un resultado incorrecto.
- `real :: pi_aprox = 3.14159265`: Los literales reales tienen punto decimal. Sin el punto, `3` sería un entero. Si el literal es muy grande o muy pequeño, se usa notación científica: `1.0e5` para 100000.0.
- `double precision :: constante_planck = 6.62607015d-34`: `double precision` es un real con mayor precisión (normalmente 8 bytes). El sufijo `d` en `d-34` indica literal de doble precisión. Con `e` sería precisión simple, y perderíamos la mitad de los dígitos.
- `complex :: z = (1.0, 2.0)`: Los complejos se construyen con paréntesis y coma: `(parte_real, parte_imaginaria)`. No hay un literal directo como en Python (que usa `j`). Si se omiten los paréntesis y se escribe `1.0, 2.0`, el compilador interpreta dos valores separados.
- `character(len=40) :: nombre = 'Ada Lovelace'`: Asigna un valor inicial a la cadena. Internamente se almacena como 'Ada Lovelace' seguido de 28 espacios hasta llenar 40.
- `logical :: es_cientifica = .true.`: Los valores lógicos siempre llevan puntos alrededor: `.true.` y `.false.`. Sin los puntos, `true` sería interpretado como una variable no declarada.
- `w = z * (0.0, 1.0)`: Multiplica el complejo `z` por la unidad imaginaria `i`. Esto equivale a una rotación de 90 grados en el plano complejo. Sin `(0.0, 1.0)`, la multiplicación sería por un escalar real.
- `real(z)`, `aimag(z)`, `abs(z)`: Funciones intrínsecas. `real()` extrae la parte real, `aimag()` la imaginaria (nótese que no es `imag` sino `aimag`), `abs()` calcula la magnitud. En Fortran 2008+, hay también `conjg(z)` para el conjugado.
- `atan2(aimag(z), real(z))`: Calcula la fase del número complejo. `atan2(y, x)` devuelve el arco tangente considerando el cuadrante. Nota: el argumento y (imaginaria) va primero, al revés de la convención matemática usual.
- `len_trim(nombre)`: Devuelve la longitud del texto sin contar espacios finales. Sin `_trim`, `len()` devolvería 40 (el tamaño declarado).
- `if (es_cientifica) then`: La condición del `if` debe ser una expresión lógica. En Fortran, no hay conversión implícita de enteros a booleanos: no se puede escribir `if (1)` como en C.

**Salida esperada**:
```
 === DEMOSTRACIÓN DE TIPOS BÁSICOS ===

 Edad   (integer):           30
 Población (integer*8): 8000000000
 Temperatura    (real):    36.5000000
 Pi aproximado  (real):    3.14159274
 Constante Planck (d.p.):   6.6260701500000003E-034
 Número complejo (z):    (1.00000000,2.00000000)
 z * i  (rotación):   (-2.00000000,1.00000000)
 Parte real de z:        1.00000000
 Parte imaginaria:        2.00000000
 Módulo (magnitud):        2.23606801
 Argumento (fase):        1.10714877
 Longitud del nombre:           12
 Ada Lovelace  es una científica.
```

**Errores típicos**:

1. **Error: `double precision` sin literal `d`**
```fortran
program error_precision
    implicit none
    double precision :: x
    x = 3.141592653589793   ! Error: literal en precisión simple
    print *, x
end program error_precision
```
El problema es que `3.141592653589793` es un literal `real` (precisión simple, ~7 dígitos), y al asignarse a `double precision` no se ganan más dígitos de los que ya trae el literal. Mensaje real: No hay error de compilación, pero el valor almacenado tiene solo 7 dígitos significativos. Solución: usar sufijo `d0`: `x = 3.141592653589793d0`.

2. **Error: punto decimal omitido en literal real**
```fortran
program error_real
    implicit none
    real :: x
    x = 5 / 2   ! Esto es división entera: resultado 2, no 2.5
    print *, x
end program error_real
```
Mensaje real: No hay error de compilación, pero `x` vale 2.0 en lugar de 2.5. La división entre dos enteros produce un entero (truncado). Solución: `x = 5.0 / 2.0` o `x = 5 / 2.0`.

**Mini-proyecto: Calculadora de conversiones físicas**

Este programa pide al usuario una temperatura en grados Celsius y la convierte a Fahrenheit, Kelvin y Rankine. Además, demuestra el uso de diferentes tipos de datos: `real` para las temperaturas, `character` para la unidad, `logical` para controlar un bucle de repetición.

```fortran
program conversor_temperatura
    implicit none

    real :: celsius, fahrenheit, kelvin, rankine
    character(len=1) :: opcion
    logical :: continuar = .true.

    do while (continuar)
        ! Encabezado
        write(*, *) '=== CONVERSOR DE TEMPERATURAS ==='
        write(*, *) 'Ingrese la temperatura en grados Celsius:'
        read(*, *) celsius

        ! Conversiones
        fahrenheit = celsius * 9.0 / 5.0 + 32.0
        kelvin     = celsius + 273.15
        rankine    = celsius * 9.0 / 5.0 + 491.67

        ! Mostrar resultados
        write(*, *)
        write(*, *) 'Resultados:'
        write(*, *) '  Celsius:    ', celsius,    ' °C'
        write(*, *) '  Fahrenheit: ', fahrenheit, ' °F'
        write(*, *) '  Kelvin:     ', kelvin,     ' K'
        write(*, *) '  Rankine:    ', rankine,    ' °R'
        write(*, *)

        ! Preguntar si desea continuar
        write(*, *) '¿Realizar otra conversión? (s/n):'
        read(*, *) opcion
        if (opcion == 'n' .or. opcion == 'N') then
            continuar = .false.
        end if
    end do

    write(*, *) '¡Gracias por usar el conversor!'

end program conversor_temperatura
```

**Análisis línea por línea**:
- `real :: celsius, fahrenheit, kelvin, rankine`: Declara cuatro variables reales en una sola línea. En Fortran se pueden agrupar variables del mismo tipo separadas por comas. No es necesario declararlas individualmente, lo que hace el código más compacto.
- `do while (continuar)`: Bucle que se ejecuta mientras la condición sea verdadera. A diferencia de `do` con contador, este bucle no tiene una variable de iteración. Si la condición es falsa desde el principio, el bucle no se ejecuta nunca.
- `celsius * 9.0 / 5.0 + 32.0`: Fórmula de conversión. Nótese que usamos `9.0` y `5.0` en lugar de `9` y `5`. Si usáramos enteros, la división `9/5` daría `1` (división entera), arruinando el cálculo.
- `read(*, *) opcion`: Lee un solo carácter desde la terminal. Si el usuario escribe "no", solo se leería el primer carácter 'n'. Esto es intencional: queremos solo la primera letra.
- `if (opcion == 'n' .or. opcion == 'N')`: Compara con minúscula y mayúscula. En Fortran, `==` es el operador de igualdad (en Fortran 77 se usaba `.eq.`). La doble comparación es necesaria porque las cadenas en Fortran distinguen entre mayúsculas y minúsculas.
- `continuar = .false.`: Cambia el flag para salir del bucle. Sin esta línea, el bucle sería infinito. Alternativamente, podríamos usar `exit` para salir inmediatamente.

**Salida esperada** (interactiva):
```
 === CONVERSOR DE TEMPERATURAS ===
 Ingrese la temperatura en grados Celsius:
> 100

 Resultados:
   Celsius:        100.000000     °C
   Fahrenheit:     212.000000     °F
   Kelvin:         373.150000     K
   Rankine:        671.670000     °R

 ¿Realizar otra conversión? (s/n):
> n
 ¡Gracias por usar el conversor!
```

---

## 3. Arrays (estáticos y dinámicos)

Imaginemos que tenemos que gestionar las calificaciones de 1000 estudiantes. Podríamos crear 1000 variables individuales: `nota1, nota2, nota3, ...`, pero eso sería una locura. Necesitamos una estructura de datos que agrupe múltiples valores del mismo tipo bajo un mismo nombre: un array. Los arrays son la columna vertebral de la computación científica, porque permiten representar vectores, matrices, tensores y cualquier colección homogénea de datos sobre la que aplicar operaciones masivas.

Fortran tiene una relación especial con los arrays. Mientras que en C un array no es más que un puntero al primer elemento, en Fortran un array es un objeto de primera clase con información sobre su forma (shape), tamaño (size) y dimensión (rank). El compilador sabe exactamente cuántos elementos tiene y cómo están organizados en memoria. Esta información permite al compilador generar código optimizado usando instrucciones SIMD (Single Instruction, Multiple Data) del procesador, y también permite al programador escribir operaciones vectoriales completas en una sola línea: `a = b + c` suma dos arrays enteros elemento a elemento, sin necesidad de bucles explícitos.

Hay dos grandes categorías de arrays en Fortran: estáticos y dinámicos. Los arrays estáticos tienen un tamaño fijo conocido en tiempo de compilación. Se declaran con `dimension(tamaño)` y la memoria se reserva en la pila (stack). Son rápidos y seguros, pero inflexibles. Los arrays dinámicos se declaran con el atributo `allocatable` y su tamaño se decide en tiempo de ejecución mediante `allocate()` y se libera con `deallocate()`. Se almacenan en el montón (heap), lo que permite manejar conjuntos de datos cuyo tamaño no conocemos hasta que el programa se ejecuta.

La decisión de usar array estático o dinámico no es meramente técnica: es filosófica. Los arrays estáticos nos protegen de errores de memoria porque el compilador verifica en tiempo de compilación que no excedamos los límites (si usamos `-fcheck=bounds`). Los arrays dinámicos nos dan flexibilidad pero requieren que el programador gestione explícitamente la memoria, lo que introduce la posibilidad de fugas de memoria si olvidamos `deallocate`. En la práctica científica, la mayoría de los arrays son dinámicos porque raramente sabemos el tamaño exacto de nuestros datos al escribir el código.

Una característica única de Fortran es la indexación por defecto comienza en 1, no en 0 como en C. Esto puede ser confuso al principio, pero es más natural para los humanos (el "primer elemento" es el 1, no el 0). Además, Fortran permite definir rangos de índice arbitrarios: `integer :: arr(-5:5)` crea un array con índices de -5 a 5 (11 elementos). Esto es enormemente útil cuando modelamos sistemas físicos con coordenadas negativas, o cuando queremos que el índice tenga significado físico.

Fortran también soporta operaciones sobre secciones de arrays (array slicing). Podemos extraer subconjuntos de un array con una sintaxis de rangos: `arr(3:7)` devuelve los elementos del 3 al 7. Podemos usar pasos: `arr(1:10:2)` devuelve los elementos impares. Esta capacidad de manipular subarrays sin bucles es una de las características más poderosas y más infravaloradas de Fortran moderno.

Para arrays dinámicos, el uso de `allocatable` es preferible al uso de punteros (`pointer`). Aunque ambos permiten memoria dinámica, `allocatable` es más seguro: el compilador libera automáticamente la memoria cuando la variable sale de ámbito (si usamos Fortran 2003+), mientras que los punteros pueden crear fugas de memoria si no los gestionamos explícitamente. La regla de oro en Fortran moderno es: usa `allocatable` siempre que puedas, y reserva `pointer` para cuando necesites estructuras de datos con referencias compartidas.

Finalmente, los arrays en Fortran se almacenan en memoria en orden column-major. Esto significa que en una matriz bidimensional `a(3,2)`, se recorren primero las filas de la primera columna, luego las filas de la segunda columna. Esto es justo al revés que C (row-major). Conocer esta diferencia es crucial para la eficiencia: si recorremos una matriz grande por filas (primer índice variando más rápido), el rendimiento puede ser hasta 10 veces peor que recorriéndola por columnas, debido a la localidad de referencia en la caché del procesador.

### Arrays estáticos

```fortran
program arrays_estaticos
    implicit none

    ! Array unidimensional de 5 enteros (índices 1 a 5)
    integer, dimension(5) :: notas
    ! Array con índices personalizados (de 0 a 10)
    integer :: contadores(0:10)
    ! Matriz 3x4 (3 filas, 4 columnas)
    real :: matriz(3, 4)
    ! Array tridimensional
    integer :: cubo(2, 3, 4)

    integer :: i, j

    ! Inicialización elemento por elemento
    notas(1) = 85
    notas(2) = 92
    notas(3) = 78
    notas(4) = 95
    notas(5) = 88

    ! Inicialización con constructor de array (Fortran 95+)
    contadores = [1, 3, 5, 7, 9, 11, 13, 15, 17, 19, 21]

    ! Inicialización con bucle anidado para matriz
    do i = 1, 3
        do j = 1, 4
            matriz(i, j) = real(i * j)
        end do
    end do

    ! Operaciones vectoriales (sin bucle explícito)
    notas = notas + 5   ! Suma 5 a todos los elementos

    ! Mostrar resultados
    write(*, *) 'Notas (incrementadas en 5):'
    write(*, *) notas

    write(*, *) 'Contadores (índice 5): ', contadores(5)

    write(*, *) 'Matriz 3x4:'
    do i = 1, 3
        write(*, *) matriz(i, :)
    end do

    ! Propiedades intrínsecas del array
    write(*, *) 'Tamaño de notas: ', size(notas)
    write(*, *) 'Dimensiones de matriz: ', shape(matriz)
    write(*, *) 'Rango de cubo: ', rank(cubo)
    write(*, *) 'Límite inferior: ', lbound(notas)
    write(*, *) 'Límite superior: ', ubound(notas)

end program arrays_estaticos
```

**Análisis línea por línea**:
- `integer, dimension(5) :: notas`: Declara un array unidimensional de 5 enteros. Sin `dimension(5)`, `notas` sería un entero simple. El atributo `dimension` puede ir antes de `::` o después del nombre con paréntesis.
- `integer :: contadores(0:10)`: Forma alternativa de declaración. Aquí `(0:10)` va después del nombre. Los índices van de 0 a 10 inclusivo (11 elementos). El primer índice por defecto es 1, pero aquí lo cambiamos a 0 explícitamente.
- `real :: matriz(3, 4)`: Array bidimensional (matriz). Primer argumento: filas (3), segundo: columnas (4). En memoria se almacena por columnas (column-major).
- `contadores = [1, 3, 5, 7, 9, 11, 13, 15, 17, 19, 21]`: Constructor de array usando corchetes `[...]`. En Fortran 77 se usaba `(/.../)`, pero los corchetes son más legibles. Ambos son válidos.
- `matriz(i, j) = real(i * j)`: Llena la matriz elemento a elemento. `real(i*j)` convierte el producto entero a real. Si no se convierte, el compilador podría truncar o emitir una advertencia.
- `notas = notas + 5`: Operación vectorial. Suma 5 a cada elemento del array. No hay bucle explícito; el compilador genera el código óptimo.
- `write(*, *) notas`: Escribe todos los elementos del array en una línea. El formato libre muestra los elementos separados por espacios.
- `write(*, *) matriz(i, :)`: El operador `:` (dos puntos) es una sección (slice). `matriz(i, :)` selecciona toda la fila `i`. Sin los dos puntos, `matriz(i)` sería ambiguo y el compilador podría dar error.
- `size(notas)`: Función intrínseca que devuelve el número total de elementos.
- `shape(matriz)`: Devuelve un array con las dimensiones: `[3, 4]` para una matriz 3x4.
- `rank(cubo)`: Devuelve el número de dimensiones (3 para un array tridimensional).
- `lbound(notas)`, `ubound(notas)`: Devuelven los límites inferior y superior del array.

**Salida esperada**:
```
 Notas (incrementadas en 5):
          90          97          83         100          93
 Contadores (índice 5):           11
 Matriz 3x4:
   1.00000000       2.00000000       3.00000000       4.00000000
   2.00000000       4.00000000       6.00000000       8.00000000
   3.00000000       6.00000000       9.00000000      12.00000000
 Tamaño de notas:            5
 Dimensiones de matriz:            3           4
 Rango de cubo:            3
 Límite inferior:            1
 Límite superior:            5
```

### Arrays dinámicos (allocatable)

```fortran
program arrays_dinamicos
    implicit none

    integer :: n
    integer, allocatable, dimension(:) :: arr      ! Array dinámico 1D
    real, allocatable :: matriz(:, :)               ! Array dinámico 2D
    integer :: i, j, istat

    ! Preguntar tamaño al usuario
    write(*, *) '¿Cuántos elementos desea?'
    read(*, *) n

    ! Reservar memoria: allocate
    allocate(arr(n), stat=istat)
    if (istat /= 0) then
        write(*, *) 'Error: no se pudo asignar memoria.'
        stop
    end if

    ! Llenar el array
    do i = 1, n
        arr(i) = i * 2
    end do

    ! Mostrar
    write(*, *) 'Array generado:'
    write(*, *) arr

    ! Reservar matriz dinámica
    allocate(matriz(3, n), stat=istat)
    if (istat /= 0) then
        write(*, *) 'Error al asignar matriz.'
        stop
    end if

    ! Llenar matriz
    do i = 1, 3
        do j = 1, n
            matriz(i, j) = real(i + j)
        end do
    end do

    ! Mostrar matriz por filas
    write(*, *) 'Matriz 3x', n, ':'
    do i = 1, 3
        write(*, *) matriz(i, :)
    end do

    ! Liberar memoria (opcional con Fortran 2003+ al salir del ámbito)
    deallocate(arr, matriz)

    ! Verificar que se liberó
    if (.not. allocated(arr)) then
        write(*, *) 'Memoria liberada correctamente.'
    end if

end program arrays_dinamicos
```

**Análisis línea por línea**:
- `integer, allocatable, dimension(:) :: arr`: Declara un array entero dinámico unidimensional. El `(:)` indica que es un array de rango 1, pero sin tamaño fijo. Sin `allocatable`, el compilador esperaría un tamaño fijo.
- `allocate(arr(n), stat=istat)`: Solicita memoria para `n` elementos. `stat=istat` captura el código de estado: 0 significa éxito, cualquier otro valor indica error (memoria insuficiente, por ejemplo). Sin `stat`, un fallo de asignación causaría un crash del programa.
- `if (istat /= 0)`: Verifica si la asignación falló. `/=0` significa "distinto de 0". En Fortran 77 se usaba `.ne.`. Si no se maneja este error, un `allocate` fallido detiene el programa con un mensaje de runtime.
- `allocate(matriz(3, n), stat=istat)`: Asigna una matriz dinámica bidimensional. La primera dimensión es fija (3), la segunda es dinámica (`n`). En Fortran se puede mezclar tamaños fijos y dinámicos en diferentes dimensiones.
- `deallocate(arr, matriz)`: Libera la memoria. Desde Fortran 2003, un `allocatable` se desasigna automáticamente al salir del ámbito, pero es buena práctica hacerlo explícitamente. Si se olvida el `deallocate`, no hay fuga de memoria si la variable está declarada con `allocatable` (el compilador la libera al final del programa).
- `allocated(arr)`: Función intrínseca que devuelve `.true.` si el array está actualmente asignado. Útil para verificar que no intentemos acceder a un array sin asignar.

**Salida esperada** (interactiva):
```
 ¿Cuántos elementos desea?
> 5
 Array generado:
           2           4           6           8          10
 Matriz 3x5:
   2.00000000       3.00000000       4.00000000       5.00000000       6.00000000
   3.00000000       4.00000000       5.00000000       6.00000000       7.00000000
   4.00000000       5.00000000       6.00000000       7.00000000       8.00000000
 Memoria liberada correctamente.
```

**Errores típicos**:

1. **Error: usar un array allocatable sin asignar memoria**
```fortran
program error_alloc
    implicit none
    integer, allocatable :: arr(:)
    arr(1) = 10   ! Error: arr no está asignado
end program error_alloc
```
Mensaje real: `Fortran runtime error: Index '1' of dimension 1 of array 'arr' above upper bound of 0`. El programa compila pero falla en ejecución. Solución: siempre verificar con `allocated(arr)` antes de usar, o asegurarse de haber llamado `allocate`.

2. **Error: acceso fuera de límites**
```fortran
program error_bounds
    implicit none
    integer :: arr(5)
    arr(10) = 42  ! Error: índice 10 fuera de rango
end program error_bounds
```
Mensaje real (con -fcheck=bounds): `Fortran runtime error: Index '10' of dimension 1 of array 'arr' above upper bound of 5`. Sin `-fcheck=bounds`, no hay error de compilación ni de runtime, y se corrompe memoria silenciosamente. Solución: compilar siempre con `-fcheck=bounds` en desarrollo y usar `lbound`/`ubound` para verificar.

**Mini-proyecto: Producto escalar con arrays allocatable**

Este programa lee dos vectores del usuario, calcula su producto escalar (suma de productos elemento a elemento) y muestra el resultado usando arrays dinámicos.

```fortran
program producto_escalar
    implicit none

    integer :: n, i
    real, allocatable :: v1(:), v2(:)
    real :: resultado = 0.0
    integer :: istat

    ! Leer dimensión
    write(*, *) '=== PRODUCTO ESCALAR DE DOS VECTORES ==='
    write(*, *) 'Ingrese la dimensión de los vectores:'
    read(*, *) n

    ! Validar entrada
    if (n <= 0) then
        write(*, *) 'Error: la dimensión debe ser positiva.'
        stop
    end if

    ! Asignar memoria
    allocate(v1(n), v2(n), stat=istat)
    if (istat /= 0) then
        write(*, *) 'Error: no se pudo asignar memoria.'
        stop
    end if

    ! Leer primer vector
    write(*, *) 'Ingrese los ', n, ' elementos del primer vector:'
    do i = 1, n
        write(*, *) 'v1(', i, ') = '
        read(*, *) v1(i)
    end do

    ! Leer segundo vector
    write(*, *) 'Ingrese los ', n, ' elementos del segundo vector:'
    do i = 1, n
        write(*, *) 'v2(', i, ') = '
        read(*, *) v2(i)
    end do

    ! Calcular producto escalar
    do i = 1, n
        resultado = resultado + v1(i) * v2(i)
    end do

    ! Mostrar vectores y resultado
    write(*, *) '---'
    write(*, *) 'v1 = ', v1
    write(*, *) 'v2 = ', v2
    write(*, *) 'Producto escalar (v1 · v2) = ', resultado

    ! Liberar memoria
    deallocate(v1, v2)

end program producto_escalar
```

**Análisis línea por línea**:
- `if (n <= 0)`: Validación de entrada del usuario. Sin esta validación, el usuario podría ingresar 0 o un número negativo, causando que `allocate(v1(0))` falle o se comporte de forma impredecible.
- `allocate(v1(n), v2(n), stat=istat)`: Asigna ambos vectores en una sola llamada. Es más eficiente que dos `allocate` separados.
- `v1(i) * v2(i)`: Multiplicación elemento a elemento. El producto escalar acumula la suma de estos productos.
- `resultado = resultado + v1(i) * v2(i)`: Acumulador. Sin la inicialización previa `resultado = 0.0`, la variable contendría un valor basura (cualquier número que estuviera en esa posición de memoria).
- `deallocate(v1, v2)`: Libera ambos arrays. Opcional en este caso porque el programa termina, pero es buena práctica para detectar fugas de memoria en programas más complejos.

**Salida esperada** (interactiva):
```
 === PRODUCTO ESCALAR DE DOS VECTORES ===
 Ingrese la dimensión de los vectores:
> 3
 Ingrese los 3 elementos del primer vector:
 v1(1) =
> 1.0
 v1(2) =
> 2.0
 v1(3) =
> 3.0
 Ingrese los 3 elementos del segundo vector:
 v2(1) =
> 4.0
 v2(2) =
> 5.0
 v2(3) =
> 6.0
 ---
 v1 =    1.00000000       2.00000000       3.00000000
 v2 =    4.00000000       5.00000000       6.00000000
 Producto escalar (v1 · v2) =    32.0000000
```

---

## 4. Control de flujo

El control de flujo es el sistema nervioso de un programa: decide qué caminos tomar basándose en condiciones y repite operaciones hasta que se cumple un criterio. Sin control de flujo, un programa sería una secuencia lineal de instrucciones que se ejecutan siempre en el mismo orden, como una receta de cocina sin decisiones ni bucles. En la vida real, los programas deben tomar decisiones: "si la temperatura supera cierto umbral, activa la alarma; si no, sigue monitoreando". También deben repetir tareas: "calcula la trayectoria para cada paso de tiempo hasta que el cohete alcance la órbita".

Fortran ofrece tres estructuras de control fundamentales: `if` para decisiones binarias o multi-rama, `select case` para selección entre múltiples valores discretos, y `do` para bucles. Cada una tiene su nicho, pero todas comparten una filosofía común: la estructura debe ser explícita y claramente delimitada. No hay llaves `{}` como en C; en su lugar, se usan palabras clave como `then`, `end if`, `select case`, `end select`, `do`, `end do`. Esta elección de diseño hace que el código sea más verboso pero también más legible: nunca hay duda de dónde empieza y termina un bloque.

El `if` de Fortran es más rico de lo que parece a simple vista. Tenemos `if` simple (`if (cond) instruccion`), `if-then-end if` para un bloque, `if-then-else-end if` para dos caminos, y `if-then-else if-else-end if` para múltiples caminos. Además, Fortran 95 introdujo el `if` con nombre (usando `if: block ... end if if`), que permite anidar estructuras complejas con claridad. El `if` en Fortran evalúa una expresión lógica: no hay conversión implícita de enteros, como en C (`if(x)`).

El `select case` es el equivalente de `switch` en C, pero con diferencias importantes. En C, cada `case` necesita un `break` explícito; si lo olvidamos, la ejecución "cae" al siguiente caso (fall-through). En Fortran, cada bloque `case` termina automáticamente sin necesidad de `break`. Además, Fortran permite rangos en los casos: `case (1:10)` ejecuta el bloque si el selector está entre 1 y 10. El `select case` solo funciona con tipos enteros, caracteres o lógicos (no con reales).

Los bucles `do` son el caballo de batalla del cómputo numérico. Un `do` simple itera sobre un rango: `do i = inicio, fin, paso`. El paso es opcional (por defecto es 1). Pero hay más variantes: `do while (cond)` itera mientras la condición sea verdadera, y el `do` infinito `do ... end do` solo termina con `exit` o `return`. Fortran 2008 introdujo `do concurrent`, una variante que indica al compilador que las iteraciones son independientes y pueden ejecutarse en paralelo.

Un aspecto sutil de los bucles `do` en Fortran es que el límite superior se evalúa solo una vez, al inicio del bucle. Si modificamos la variable que sirve como límite dentro del bucle, esto no afecta al número de iteraciones. Esto es diferente de C, donde la condición de continuación se evalúa en cada iteración. Además, en Fortran el índice del bucle (`i`) no debe modificarse manualmente dentro del bucle; hacerlo es ilegal en algunos estándares y produce comportamiento indefinido.

Las sentencias `exit` y `cycle` permiten control fino dentro de los bucles. `exit` sale inmediatamente del bucle, mientras que `cycle` salta a la siguiente iteración. Ambas deben usarse con moderación: un uso excesivo de `exit` y `cycle` puede convertir un bucle limpio en un espagueti de control. A menudo, un `do while` bien diseñado elimina la necesidad de `exit`.

### if, select case, do

```fortran
program control_flujo
    implicit none

    integer :: opcion, i, n
    integer :: suma = 0
    real :: nota
    logical :: es_par

    write(*, *) '=== CONTROL DE FLUJO EN FORTRAN ==='
    write(*, *)
    write(*, *) '1. Ejemplo de IF'
    write(*, *) '2. Ejemplo de SELECT CASE'
    write(*, *) '3. Ejemplo de DO loops'
    write(*, *) '4. Salir'
    write(*, *)
    write(*, *) 'Seleccione una opción:'
    read(*, *) opcion

    ! --- SELECT CASE externo ---
    select case (opcion)

        ! --- BLOQUE 1: IF ---
        case (1)
            write(*, *) '--- EJEMPLO IF ---'
            write(*, *) 'Ingrese una nota (0-100):'
            read(*, *) nota

            if (nota < 0.0 .or. nota > 100.0) then
                write(*, *) 'Error: nota fuera de rango.'
            else if (nota >= 90.0) then
                write(*, *) 'Calificación: A (Excelente)'
            else if (nota >= 80.0) then
                write(*, *) 'Calificación: B (Bueno)'
            else if (nota >= 70.0) then
                write(*, *) 'Calificación: C (Suficiente)'
            else if (nota >= 60.0) then
                write(*, *) 'Calificación: D (Deficiente)'
            else
                write(*, *) 'Calificación: F (Reprobado)'
            end if

            ! Verificar si es par (usando modulo)
            es_par = (mod(int(nota), 2) == 0)
            if (es_par) then
                write(*, *) 'La nota es un número par.'
            else
                write(*, *) 'La nota es un número impar.'
            end if

        ! --- BLOQUE 2: SELECT CASE ---
        case (2)
            write(*, *) '--- EJEMPLO SELECT CASE ---'
            write(*, *) 'Ingrese un número de mes (1-12):'
            read(*, *) n

            select case (n)
                case (1)
                    write(*, *) 'Enero: 31 días'
                case (2)
                    write(*, *) 'Febrero: 28 o 29 días'
                case (3)
                    write(*, *) 'Marzo: 31 días'
                case (4)
                    write(*, *) 'Abril: 30 días'
                case (5)
                    write(*, *) 'Mayo: 31 días'
                case (6)
                    write(*, *) 'Junio: 30 días'
                case (7)
                    write(*, *) 'Julio: 31 días'
                case (8)
                    write(*, *) 'Agosto: 31 días'
                case (9)
                    write(*, *) 'Septiembre: 30 días'
                case (10)
                    write(*, *) 'Octubre: 31 días'
                case (11)
                    write(*, *) 'Noviembre: 30 días'
                case (12)
                    write(*, *) 'Diciembre: 31 días'
                case default
                    write(*, *) 'Error: mes inválido (debe ser 1-12).'
            end select

            ! Segundo ejemplo: rangos en case
            write(*, *) '--- Rangos en CASE ---'
            write(*, *) 'Ingrese un número (1-100):'
            read(*, *) n

            select case (n)
                case (1:30)
                    write(*, *) 'El número está en el rango [1, 30].'
                case (31:60)
                    write(*, *) 'El número está en el rango [31, 60].'
                case (61:100)
                    write(*, *) 'El número está en el rango [61, 100].'
                case default
                    write(*, *) 'Número fuera del rango 1-100.'
            end select

        ! --- BLOQUE 3: DO loops ---
        case (3)
            write(*, *) '--- EJEMPLO DO LOOPS ---'

            ! Bucle simple con contador
            write(*, *) 'Bucle simple (1 a 5):'
            do i = 1, 5
                write(*, *) '  Iteración ', i
            end do

            ! Bucle con paso
            write(*, *) 'Bucle con paso 3 (0 a 12):'
            do i = 0, 12, 3
                write(*, *) '  i = ', i
            end do

            ! Bucle descendente (paso negativo)
            write(*, *) 'Bucle descendente (10 a 1):'
            do i = 10, 1, -1
                write(*, *) '  i = ', i
            end do

            ! Bucle do while
            write(*, *) 'Bucle DO WHILE (sumar hasta superar 100):'
            suma = 0
            i = 1
            do while (suma <= 100)
                suma = suma + i
                i = i + 1
            end do
            write(*, *) '  Suma final: ', suma
            write(*, *) '  Números sumados: ', i - 1

            ! Bucle con exit y cycle
            write(*, *) 'Bucle con EXIT y CYCLE:'
            do i = 1, 20
                if (mod(i, 5) == 0) then
                    cycle   ! Saltar múltiplos de 5
                end if
                if (i > 15) then
                    exit    ! Salir del bucle si i > 15
                end if
                write(*, *) '  i = ', i
            end do

            ! Bucle anidado
            write(*, *) 'Bucle anidado (tabla de multiplicar 3x3):'
            do i = 1, 3
                do n = 1, 3
                    write(*, '(I2, " x ", I2, " = ", I2)') i, n, i * n
                end do
            end do

        ! --- OPCIÓN POR DEFECTO ---
        case default
            if (opcion /= 4) then
                write(*, *) 'Error: opción no válida.'
            end if

    end select

    if (opcion == 4) then
        write(*, *) '¡Hasta luego!'
    end if

end program control_flujo
```

**Análisis línea por línea**:
- `select case (opcion)`: Comienza un bloque de selección múltiple. La variable `opcion` debe ser entera, carácter o lógica. No puede ser `real`. Cada rama `case` evalúa si coincide con el valor de `opcion`.
- `case (1)`: Rama que se ejecuta si `opcion == 1`. No lleva `:` después del valor (en C sería `case 1:`). No necesita `break`; la ejecución no "cae" al siguiente caso.
- `case default`: Rama opcional que se ejecuta si ningún otro `case` coincide. Debe ir al final. Sin `case default`, si ningún caso coincide, simplemente no se ejecuta nada.
- `if (nota < 0.0 .or. nota > 100.0) then`: Condición compuesta con `.or.` (O lógico). También se pueden usar `.and.` (Y), `.not.` (NO). En Fortran 77 se usaban operadores como `.lt.` en lugar de `<`, pero los símbolos son válidos desde Fortran 90.
- `es_par = (mod(int(nota), 2) == 0)`: `mod(a,b)` da el resto de `a/b`. Si el resto es 0, es par. `int(nota)` convierte el real a entero (truncando). Si la nota fuera 89.9, `int` devuelve 89.
- `do i = 0, 12, 3`: Bucle for-style. Comienza en 0, termina en 12 inclusive, incrementa en 3 cada iteración. El paso (3) es el tercer argumento. Si se omite, el paso es 1.
- `do i = 10, 1, -1`: Bucle descendente. Comienza en 10, termina en 1 (no en 0), decrementa en 1. El límite inferior (1) es menor que el superior (10), pero el paso es negativo.
- `do while (suma <= 100)`: Bucle condicional. Se ejecuta mientras `suma` sea <= 100. La condición se evalúa al inicio de cada iteración. Si la condición es falsa desde el principio, el bucle no se ejecuta.
- `if (mod(i, 5) == 0) then ... cycle`: `cycle` salta al final del bloque `do` y comienza la siguiente iteración. Es como `continue` en C. Sin `cycle`, se ejecutaría el resto del bloque.
- `if (i > 15) then ... exit`: `exit` sale inmediatamente del bucle, continuando después de `end do`. Sin `exit`, el bucle continuaría hasta su fin natural.

**Salida esperada** (opción 3):
```
 --- EJEMPLO DO LOOPS ---
 Bucle simple (1 a 5):
   Iteración            1
   Iteración            2
   Iteración            3
   Iteración            4
   Iteración            5
 Bucle con paso 3 (0 a 12):
   i =            0
   i =            3
   i =            6
   i =            9
   i =           12
 Bucle descendente (10 a 1):
   i =           10
   i =            9
   i =            8
   i =            7
   i =            6
   i =            5
   i =            4
   i =            3
   i =            2
   i =            1
 Bucle DO WHILE (sumar hasta superar 100):
   Suma final:          105
   Números sumados:           14
 Bucle con EXIT y CYCLE:
   i =            1
   i =            2
   i =            3
   i =            4
   i =            6
   i =            7
   i =            8
   i =            9
   i =           11
   i =           12
   i =           13
   i =           14
   i =           15
 Bucle anidado (tabla de multiplicar 3x3):
  1 x  1 =  1
  1 x  2 =  2
  1 x  3 =  3
  2 x  1 =  2
  2 x  2 =  4
  2 x  3 =  6
  3 x  1 =  3
  3 x  2 =  6
  3 x  3 =  9
```

**Errores típicos**:

1. **Error: condición `if` mal formada**
```fortran
program error_if
    implicit none
    integer :: x = 5
    if (x) then  ! Error: x no es lógico, es entero
        print *, 'verdadero'
    end if
end program error_if
```
Mensaje real: `Error: IF clause at (1) requires a scalar LOGICAL expression`. En C esto sería válido (cualquier valor distinto de 0 es verdadero), pero Fortran exige una expresión explícitamente lógica. Solución: `if (x /= 0) then`.

2. **Error: `select case` con tipo real**
```fortran
program error_case
    implicit none
    real :: x = 2.5
    select case (x)  ! Error: real no permitido
        case (1.0)
            print *, 'uno'
    end select
end program error_case
```
Mensaje real: `Error: SELECT CASE at (1) expression must be of a discrete type (Integer, Logical or CHARACTER)`. Los reales no son discretos, por lo que no pueden usarse en `select case`. Solución: usar `if-else if-else` para rangos reales.

**Mini-proyecto: Juego de adivinanza numérica**

Un programa que genera un número aleatorio entre 1 y 100, y el usuario debe adivinarlo. El programa da pistas de "mayor" o "menor" y cuenta los intentos. Demostración de `do while`, `if`, `exit`, y funciones intrínsecas de aleatoriedad.

```fortran
program adivinanza
    implicit none

    integer :: secreto, intento, contador
    real :: semilla_aleatoria

    ! Inicializar generador aleatorio (Fortran 95+)
    call random_seed()

    ! Generar número aleatorio entre 1 y 100
    call random_number(semilla_aleatoria)
    secreto = int(semilla_aleatoria * 100.0) + 1

    contador = 0

    write(*, *) '=== ADIVINA EL NÚMERO SECRETO ==='
    write(*, *) 'He elegido un número entre 1 y 100.'
    write(*, *) 'Intenta adivinarlo...'
    write(*, *)

    ! Bucle principal del juego
    do
        write(*, *) 'Tu intento: '
        read(*, *) intento
        contador = contador + 1

        ! Comparar
        if (intento < secreto) then
            write(*, *) '¡Muy bajo! Intenta con un número mayor.'
        else if (intento > secreto) then
            write(*, *) '¡Muy alto! Intenta con un número menor.'
        else
            write(*, *) '¡Felicidades! ¡Adivinaste!'
            write(*, *) 'El número secreto era: ', secreto
            write(*, *) 'Lo lograste en ', contador, ' intentos.'
            exit
        end if
    end do

end program adivinanza
```

**Análisis línea por línea**:
- `call random_seed()`: Inicializa el generador de números aleatorios con una semilla basada en la hora del sistema. Sin esta llamada, la secuencia aleatoria es la misma cada vez que se ejecuta el programa (útil para depuración, pero no para un juego).
- `call random_number(semilla_aleatoria)`: Genera un número real aleatorio en el rango [0.0, 1.0). La variable debe ser `real`. No acepta enteros directamente.
- `int(semilla_aleatoria * 100.0) + 1`: Escala el número aleatorio al rango [0, 99], luego se suma 1 para obtener [1, 100]. `int()` trunca, no redondea.
- `do`: Bucle infinito. Termina solo con `exit` dentro del cuerpo. Peligroso si olvidamos `exit`, pero útil cuando no sabemos cuántas iteraciones se necesitan.
- `contador = contador + 1`: Incrementa el contador de intentos. En Fortran no existe el operador `+=` como en C. Hay que escribir la operación completa.
- `exit`: Sale del bucle infinito. Sin `exit`, el bucle continuaría para siempre preguntando al usuario.

**Salida esperada** (interactiva):
```
 === ADIVINA EL NÚMERO SECRETO ===
 He elegido un número entre 1 y 100.
 Intenta adivinarlo...

 Tu intento:
> 50
 ¡Muy bajo! Intenta con un número mayor.
 Tu intento:
> 75
 ¡Muy alto! Intenta con un número menor.
 Tu intento:
> 62
 ¡Muy alto! Intenta con un número menor.
 Tu intento:
> 56
 ¡Felicidades! ¡Adivinaste!
 El número secreto era:           56
 Lo lograste en            4 intentos.
```

---

## 5. Funciones y subrutinas

Cuando un programa comienza a crecer, llega un punto en que tener todo el código dentro de un único bloque `program` se vuelve una pesadilla. Mil líneas de código mezclando entrada, salida, cálculos y lógica de negocio son imposibles de leer, depurar y mantener. La solución milenaria es la misma que usamos en la vida cotidiana: dividir el trabajo en tareas más pequeñas y especializadas. En programación, esas tareas son las funciones y subrutinas.

Fortran distingue entre funciones y subrutinas, una diferenciación que otros lenguajes han borrado. Una función devuelve un valor (como una función matemática: `f(x)` produce un número), mientras que una subrutina es un bloque de código que realiza operaciones pero no devuelve un valor directamente (como un procedimiento que imprime un informe). Las funciones se invocan dentro de expresiones: `y = sqrt(x)`. Las subrutinas se invocan con `call`: `call ordenar(lista)`. Esta distinción obliga al programador a pensar si lo que necesita es un "cálculo que produce un resultado" o una "acción que modifica el estado".

Las funciones en Fortran tienen una característica peculiar: pueden declararse con `result` para nombrar explícitamente la variable de retorno. Por ejemplo: `function suma(a, b) result(r)`. Esto es útil cuando la función tiene un nombre que no es conveniente para la variable de retorno. Sin `result`, el nombre de la función actúa como la variable de retorno. En funciones recursivas, `result` es obligatorio.

El paso de argumentos en Fortran moderno es por defecto por referencia. Esto significa que cuando pasamos una variable a una función o subrutina, estamos pasando la dirección de memoria, no una copia. Cualquier modificación que hagamos al argumento dentro del subprograma se refleja en la variable original. Esto es diferente de Python (que pasa por referencia pero los tipos inmutables se comportan como paso por valor) y de C (que pasa por valor a menos que usemos punteros explícitamente).

Para proteger los argumentos que no deben ser modificados, Fortran ofrece el atributo `intent`. `intent(in)` indica que el argumento es de solo lectura, `intent(out)` que es de solo escritura, y `intent(inout)` que puede ser leído y modificado. Usar `intent` no es opcional decorativo: el compilador puede hacer optimizaciones y verificar que no estemos violando la intencionalidad declarada. Es una de las herramientas más importantes para escribir código robusto.

Las funciones pueden ser puras (`pure`) o elementales (`elemental`). Una función pura no tiene efectos secundarios: no modifica sus argumentos ni variables globales, y siempre devuelve el mismo resultado para los mismos argumentos. Una función elemental opera sobre escalares pero puede aplicarse automáticamente a arrays. Por ejemplo, si escribimos `elemental real function doble(x)` y luego llamamos `doble(mi_array)`, la función se aplica a cada elemento automáticamente.

Una palabra clave importante es `contains`. Dentro de un `program`, un `module` o otro subprograma, podemos definir funciones y subrutinas internas usando `contains`. Estos subprogramas internos tienen acceso a las variables del programa contenedor (host association), lo que puede ser útil pero también peligroso: el programador puede modificar inadvertidamente variables del programa principal. La regla general es usar `contains` para subprogramas pequeños que son parte integral de la unidad, y módulos para subprogramas reutilizables.

### Funciones y subrutinas explícitas

```fortran
program funciones_subrutinas
    implicit none

    ! Declaración de variables
    real :: a, b, resultado
    integer :: n
    real, allocatable :: datos(:)
    integer :: i

    ! ============================================
    ! Uso de funciones
    ! ============================================
    write(*, *) '=== FUNCIONES ==='
    write(*, *) 'Ingrese dos números reales:'
    read(*, *) a, b

    ! Llamada a función que devuelve un valor
    resultado = suma(a, b)
    write(*, *) 'Suma: ', resultado

    resultado = producto(a, b)
    write(*, *) 'Producto: ', resultado

    ! Función con múltiples argumentos
    resultado = hipotenusa(a, b)
    write(*, *) 'Hipotenusa: ', resultado

    ! ============================================
    ! Uso de subrutinas
    ! ============================================
    write(*, *)
    write(*, *) '=== SUBRUTINAS ==='

    ! Intercambiar valores
    write(*, *) 'Antes del intercambio: a = ', a, ', b = ', b
    call intercambiar(a, b)
    write(*, *) 'Después del intercambio: a = ', a, ', b = ', b

    ! Procesar array
    write(*, *) 'Ingrese el tamaño del array:'
    read(*, *) n
    allocate(datos(n))

    write(*, *) 'Ingrese ', n, ' números:'
    do i = 1, n
        read(*, *) datos(i)
    end do

    ! Llamar subrutina que procesa el array
    call procesar_array(datos, n)

    deallocate(datos)

    ! ============================================
    ! Función recursiva
    ! ============================================
    write(*, *)
    write(*, *) '=== FUNCIÓN RECURSIVA ==='
    write(*, *) 'Ingrese un entero para calcular factorial:'
    read(*, *) n
    write(*, *) n, '! = ', factorial(n)

! ============================================
! Contiene: definiciones de funciones y subrutinas internas
! ============================================
contains

    ! Función simple: suma dos reales
    real function suma(x, y)
        implicit none
        real, intent(in) :: x, y
        suma = x + y
    end function suma

    ! Función con nombre de resultado explícito
    function producto(x, y) result(prod)
        implicit none
        real, intent(in) :: x, y
        real :: prod
        prod = x * y
    end function producto

    ! Función que usa funciones intrínsecas
    real function hipotenusa(x, y)
        implicit none
        real, intent(in) :: x, y
        hipotenusa = sqrt(x**2 + y**2)
    end function hipotenusa

    ! Subrutina que intercambia dos valores
    subroutine intercambiar(x, y)
        implicit none
        real, intent(inout) :: x, y
        real :: temp
        temp = x
        x = y
        y = temp
    end subroutine intercambiar

    ! Subrutina que procesa un array
    subroutine procesar_array(arr, n)
        implicit none
        integer, intent(in) :: n
        real, intent(inout) :: arr(n)
        real :: suma, media
        integer :: j

        ! Calcular suma
        suma = 0.0
        do j = 1, n
            suma = suma + arr(j)
        end do
        media = suma / real(n)

        write(*, *) 'Datos ingresados: ', arr
        write(*, *) 'Suma: ', suma
        write(*, *) 'Media: ', media

        ! Normalizar: restar la media
        arr = arr - media
        write(*, *) 'Datos normalizados (x - media): ', arr

    end subroutine procesar_array

    ! Función recursiva con RESULT
    recursive function factorial(n) result(fact)
        implicit none
        integer, intent(in) :: n
        integer :: fact
        if (n <= 1) then
            fact = 1
        else
            fact = n * factorial(n - 1)
        end if
    end function factorial

end program funciones_subrutinas
```

**Análisis línea por línea**:
- `real function suma(x, y)`: Define una función que devuelve un real. Sin el tipo antes de `function`, la función devuelve el tipo predeterminado según la primera letra del nombre (por las reglas implícitas). Siempre es más seguro escribir el tipo explícitamente.
- `real, intent(in) :: x, y`: Declara que `x` e `y` son argumentos de entrada. No pueden ser modificados dentro de la función. Si accidentalmente escribimos `x = x + 1`, el compilador dará error. Sin `intent(in)`, el compilador permite la modificación.
- `suma = x + y`: Asigna el resultado al nombre de la función. El valor de `suma` es lo que se devuelve al llamar la función. Si no se asigna ningún valor, la función devuelve un valor basura.
- `function producto(x, y) result(prod)`: Versión con `result`. La variable de retorno es `prod`, no `producto`. Esto es útil cuando el nombre de la función no es un buen nombre de variable. Obligatorio para funciones recursivas.
- `real :: prod`: Declara el tipo de la variable de retorno. Sin esta declaración, `prod` heredaría el tipo de `producto`.
- `sqrt(x**2 + y**2)`: `sqrt()` es una función intrínseca de Fortran. `**` es el operador de potenciación. `x**2` es "x al cuadrado". Nota: `**` funciona con reales también: `x**0.5` es la raíz cuadrada.
- `subroutine intercambiar(x, y)`: Subrutina (no devuelve valor). No tiene tipo, no usa `=` para retornar. Se invoca con `call`.
- `real, intent(inout) :: x, y`: Los argumentos se leen y modifican. Si usáramos `intent(in)`, el compilador no permitiría `x = y` dentro.
- `temp = x; x = y; y = temp`: Algoritmo clásico de intercambio con variable temporal. Sin `temp`, sobrescribiríamos `x` con `y` y perderíamos el valor original.
- `recursive function factorial(n) result(fact)`: La palabra `recursive` es obligatoria para funciones que se llaman a sí mismas. Sin `recursive`, el compilador rechaza la auto-llamada. `result(fact)` también es obligatorio en funciones recursivas.
- `if (n <= 1) then ... fact = 1`: Caso base de la recursión. Sin caso base, la función se llamaría infinitamente hasta agotar la pila (stack overflow).

**Salida esperada** (interactiva):
```
 === FUNCIONES ===
 Ingrese dos números reales:
> 3.0 4.0
 Suma:    7.00000000
 Producto:    12.0000000
 Hipotenusa:    5.00000000

 === SUBRUTINAS ===
 Antes del intercambio: a =    3.00000000 , b =    4.00000000
 Después del intercambio: a =    4.00000000 , b =    3.00000000
 Ingrese el tamaño del array:
> 5
 Ingrese 5 números:
> 1 2 3 4 5
 Datos ingresados:    1.00000000       2.00000000       3.00000000       4.00000000       5.00000000
 Suma:    15.0000000
 Media:    3.00000000
 Datos normalizados (x - media):   -2.00000000      -1.00000000       0.00000000       1.00000000       2.00000000

 === FUNCIÓN RECURSIVA ===
 Ingrese un entero para calcular factorial:
> 6
 6! =          720
```

**Errores típicos**:

1. **Error: función recursiva sin `recursive`**
```fortran
program error_rec
    implicit none
    integer :: n = 5
    print *, fact(n)
contains
    integer function fact(n)   ! Falta 'recursive'
        integer, intent(in) :: n
        if (n <= 1) then
            fact = 1
        else
            fact = n * fact(n - 1)
        end if
    end function
end program error_rec
```
Mensaje real: `Error: Function 'fact' at (1) cannot be called recursively, as it is not marked as recursive`. Solución: añadir `recursive` antes de `function` y usar `result`.

2. **Error: argumento `intent(in)` modificado**
```fortran
program error_intent
    implicit none
    call sub(5)
contains
    subroutine sub(x)
        integer, intent(in) :: x
        x = 10  ! Error: intent(in) no permite modificar
    end subroutine
end program error_intent
```
Mensaje real: `Error: Dummy argument 'x' with INTENT(IN) in variable definition context at (1)`. Solución: cambiar a `intent(inout)` o usar una variable local.

**Mini-proyecto: Estadísticas descriptivas con funciones y subrutinas**

Un programa que calcula la media, varianza, desviación estándar, valor mínimo y máximo de un conjunto de números usando funciones y subrutinas organizadas con `contains`.

```fortran
program estadisticas_desc
    implicit none

    integer :: n, i
    real, allocatable :: datos(:)
    real :: media, varianza, desv_std, minimo, maximo
    integer :: istat

    ! Leer tamaño
    write(*, *) '=== ESTADÍSTICAS DESCRIPTIVAS ==='
    write(*, *) '¿Cuántos números va a ingresar?'
    read(*, *) n

    if (n <= 0) then
        write(*, *) 'Error: debe ingresar al menos un número.'
        stop
    end if

    allocate(datos(n), stat=istat)
    if (istat /= 0) then
        write(*, *) 'Error de asignación de memoria.'
        stop
    end if

    ! Leer datos
    write(*, *) 'Ingrese los ', n, ' números (uno por línea o separados por espacio):'
    do i = 1, n
        read(*, *) datos(i)
    end do

    ! Calcular estadísticas usando funciones y subrutinas
    media = calcular_media(datos, n)
    varianza = calcular_varianza(datos, n, media)
    desv_std = sqrt(varianza)
    call encontrar_extremos(datos, n, minimo, maximo)

    ! Mostrar resultados
    write(*, *)
    write(*, *) 'RESULTADOS:'
    write(*, *) '  Media aritmética:         ', media
    write(*, *) '  Varianza muestral:        ', varianza
    write(*, *) '  Desviación estándar:      ', desv_std
    write(*, *) '  Valor mínimo:             ', minimo
    write(*, *) '  Valor máximo:             ', maximo

    deallocate(datos)

contains

    ! Función para calcular la media
    real function calcular_media(arr, n)
        implicit none
        integer, intent(in) :: n
        real, intent(in) :: arr(n)
        real :: suma
        integer :: i
        suma = 0.0
        do i = 1, n
            suma = suma + arr(i)
        end do
        calcular_media = suma / real(n)
    end function calcular_media

    ! Función para calcular la varianza muestral
    real function calcular_varianza(arr, n, media)
        implicit none
        integer, intent(in) :: n
        real, intent(in) :: arr(n), media
        real :: suma_cuadrados
        integer :: i
        suma_cuadrados = 0.0
        do i = 1, n
            suma_cuadrados = suma_cuadrados + (arr(i) - media)**2
        end do
        calcular_varianza = suma_cuadrados / real(n - 1)
    end function calcular_varianza

    ! Subrutina para encontrar mínimo y máximo
    subroutine encontrar_extremos(arr, n, minimo, maximo)
        implicit none
        integer, intent(in) :: n
        real, intent(in) :: arr(n)
        real, intent(out) :: minimo, maximo
        integer :: i
        minimo = arr(1)
        maximo = arr(1)
        do i = 2, n
            if (arr(i) < minimo) minimo = arr(i)
            if (arr(i) > maximo) maximo = arr(i)
        end do
    end subroutine encontrar_extremos

end program estadisticas_desc
```

**Análisis línea por línea**:
- `real, intent(in) :: arr(n)`: El array se pasa con forma explícita. `n` debe coincidir con el tamaño real del array. Si pasamos menos elementos de los declarados, solo se usarán los primeros `n`.
- `suma / real(n)`: Conversión explícita a real para evitar división entera. Sin `real(n)`, si `n` es entero, la división sería entera (truncando si `suma` también es entera).
- `(arr(i) - media)**2`: Diferencia al cuadrado. El operador `**` tiene alta precedencia: primero calcula la diferencia, luego la eleva al cuadrado.
- `suma_cuadrados / real(n - 1)`: Varianza muestral (divide por n-1, no por n). Si dividiéramos por `n`, estaríamos calculando la varianza poblacional, no la muestral.
- `real, intent(out) :: minimo, maximo`: Argumentos de solo salida. La subrutina debe asignarles valores. Si no se asigna, la variable llamante queda con valor indefinido.

**Salida esperada** (interactiva):
```
 === ESTADÍSTICAS DESCRIPTIVAS ===
 ¿Cuántos números va a ingresar?
> 5
 Ingrese los 5 números (uno por línea o separados por espacio):
> 10.5
> 20.3
> 30.1
> 15.7
> 25.9

 RESULTADOS:
   Media aritmética:          20.5000000
   Varianza muestral:          59.7500000
   Desviación estándar:        7.72981387
   Valor mínimo:               10.5000000
   Valor máximo:               30.1000000
```

---

## 6. Módulos (module)

Los módulos son, probablemente, la innovación más importante que Fortran 90 trajo al lenguaje. Antes de los módulos, la forma principal de compartir código entre programas era mediante archivos de inclusión (`include`), una técnica primitiva que equivalía a copiar y pegar código fuente. Los módulos cambiaron esto al ofrecer un mecanismo formal de encapsulación, organización y reutilización de código.

Pensemos en un módulo como una biblioteca especializada. Un módulo puede contener definiciones de tipos de datos, constantes, variables, funciones, subrutinas y, en Fortran 2003+, incluso tipos derivados con métodos (programación orientada a objetos). Lo más importante es que un módulo tiene una sección pública (accesible desde fuera) y una sección privada (visible solo dentro del módulo), lo que permite ocultar detalles de implementación. Esta separación entre interfaz e implementación es fundamental para la ingeniería de software.

La sintaxis de un módulo es simple: comienza con `module nombre_modulo`, contiene las declaraciones y definiciones, y termina con `end module nombre_modulo`. Para usar un módulo en un programa o en otro módulo, se usa `use nombre_modulo`. El `use` debe ir al principio del programa, antes de `implicit none`. Además, `use` puede llevar cláusulas `only`, que limitan qué símbolos se importan, y `rename`, que cambia el nombre local de un símbolo importado.

Una de las ventajas más importantes de los módulos es que proporcionan interfaces explícitas automáticas. Cuando un programa usa un módulo que contiene una función, el compilador sabe exactamente el tipo, número e intencionalidad de los argumentos. Esto permite al compilador verificar que la llamada es correcta en tiempo de compilación, evitando errores clásicos como pasar un entero donde se espera un real. Sin módulos, tendríamos que escribir interfaces explícitas manualmente (con `interface`), lo cual es tedioso y propenso a errores.

Los módulos también resuelven el problema de las variables globales compartidas. Podemos declarar variables en un módulo (en su sección de declaraciones, fuera de cualquier subprograma) y todos los subprogramas del módulo tienen acceso a ellas. Si otros programas o módulos usan el módulo, también tienen acceso a esas variables (a menos que sean privadas). Esto permite compartir datos sin pasarlos como argumentos, pero con el peligro de que cualquier parte del código puede modificarlos.

El encapsulamiento se logra con los atributos `public` y `private`. Por defecto, todo en un módulo es público. Podemos cambiar el valor por defecto con `private` a nivel de módulo, y luego marcar explícitamente como `public` solo lo que queremos exponer. Esta práctica se llama "mínimo privilegio": solo expón lo que es estrictamente necesario. Cambiar la implementación interna se vuelve trivial si la interfaz pública no cambia.

Los módulos no son solo para programas grandes. Incluso en programas pequeños, los módulos ofrecen beneficios inmediatos: organizan el código, proveen verificación de tipos en las llamadas, evitan la duplicación de código y hacen que las dependencias sean explícitas. Mi regla personal es: cualquier programa que tenga más de tres funciones o subrutinas debería usar módulos. La inversión inicial en aprender la sintaxis de módulos se recupera múltiples veces en tiempo de depuración evitado.

### Definición y uso de módulos

```fortran
! modulo/operaciones_matematicas.f90
module operaciones_matematicas
    implicit none

    ! Constantes públicas
    real, parameter :: PI = 3.14159265358979323846
    real, parameter :: E = 2.71828182845904523536

    ! La variable 'contador' es privada al módulo
    integer, private :: contador_llamadas = 0

    ! Interfaz genérica: 'cuadrado' funciona con entero y real
    interface cuadrado
        module procedure cuadrado_entero
        module procedure cuadrado_real
    end interface cuadrado

contains

    ! Incrementar contador (accesible públicamente)
    subroutine registrar_llamada()
        implicit none
        contador_llamadas = contador_llamadas + 1
    end subroutine registrar_llamada

    ! Obtener contador (público)
    function obtener_contador() result(cnt)
        implicit none
        integer :: cnt
        cnt = contador_llamadas
    end function obtener_contador

    ! Función para calcular área del círculo
    function area_circulo(radio) result(area)
        implicit none
        real, intent(in) :: radio
        real :: area
        call registrar_llamada()
        area = PI * radio**2
    end function area_circulo

    ! Función para calcular perímetro del círculo
    function perimetro_circulo(radio) result(perim)
        implicit none
        real, intent(in) :: radio
        real :: perim
        call registrar_llamada()
        perim = 2.0 * PI * radio
    end function perimetro_circulo

    ! Versión entera de cuadrado
    integer function cuadrado_entero(x)
        implicit none
        integer, intent(in) :: x
        call registrar_llamada()
        cuadrado_entero = x * x
    end function cuadrado_entero

    ! Versión real de cuadrado
    real function cuadrado_real(x)
        implicit none
        real, intent(in) :: x
        call registrar_llamada()
        cuadrado_real = x * x
    end function cuadrado_real

end module operaciones_matematicas
```

```fortran
! modulo/uso_modulo.f90
program uso_modulo
    use operaciones_matematicas, only: PI, E, area_circulo, &
                                       perimetro_circulo, cuadrado

    implicit none

    real :: radio, area, perimetro
    integer :: valor = 5

    write(*, *) '=== USO DE MÓDULOS ==='
    write(*, *)
    write(*, *) 'Constantes del módulo:'
    write(*, *) '  PI = ', PI
    write(*, *) '  E  = ', E
    write(*, *)

    ! Calcular área y perímetro de un círculo
    write(*, *) 'Ingrese el radio del círculo:'
    read(*, *) radio

    area = area_circulo(radio)
    perimetro = perimetro_circulo(radio)

    write(*, *) 'Resultados:'
    write(*, *) '  Área:      ', area
    write(*, *) '  Perímetro: ', perimetro
    write(*, *)

    ! Usar la interfaz genérica 'cuadrado'
    write(*, *) 'Cuadrado de ', valor, ' (entero): ', cuadrado(valor)
    write(*, *) 'Cuadrado de ', radio, ' (real):   ', cuadrado(radio)

    write(*, *)
    write(*, *) 'El módulo se ha usado ', obtener_contador(), ' veces.'

end program uso_modulo
```

**Análisis línea por línea (módulo)**:
- `module operaciones_matematicas`: Inicio del módulo. Debe estar en un archivo separado o al inicio del archivo contenedor. El orden de compilación importa: el módulo debe compilarse antes que cualquier programa que lo use.
- `real, parameter :: PI = 3.14159265358979323846`: Constante (`parameter` significa que no puede cambiar). Las constantes son evaluadas en tiempo de compilación. Sin `parameter`, sería una variable modificable.
- `integer, private :: contador_llamadas = 0`: Variable privada: solo accesible dentro del módulo. Sin `private`, cualquier unidad que haga `use` podría modificar el contador.
- `interface cuadrado ... end interface`: Interfaz genérica (overloading). Define que `cuadrado` puede llamarse con argumentos enteros o reales. El compilador selecciona automáticamente la implementación correcta según el tipo del argumento.
- `module procedure cuadrado_entero`: Asocia el nombre genérico `cuadrado` con la implementación concreta. Sin esta línea, `cuadrado` no sabría qué función llamar cuando se pasa un entero.
- `contains`: Marca el inicio de los subprogramas contenidos en el módulo. Todo lo que está antes de `contains` son declaraciones; todo lo que está después son definiciones de procedimientos.

**Análisis línea por línea (programa)**:
- `use operaciones_matematicas, only: PI, E, area_circulo, perimetro_circulo, cuadrado`: Importa solo los símbolos especificados. `only` es opcional pero altamente recomendable: evita contaminar el espacio de nombres y hace explícitas las dependencias. Sin `only`, se importaría todo el contenido público del módulo.
- `call obtener_contador()`: Aunque `contador_llamadas` es privado, `obtener_contador()` es público y permite acceder al valor del contador de forma controlada.

**Salida esperada**:
```
 === USO DE MÓDULOS ===

 Constantes del módulo:
   PI =    3.14159274
   E  =    2.71828175

 Ingrese el radio del círculo:
> 2.5
 Resultados:
   Área:        19.6349545
   Perímetro:   15.7079639

 Cuadrado de            5 (entero):           25
 Cuadrado de    2.50000000 (real):      6.25000000

 El módulo se ha usado            5 veces.
```

**Errores típicos**:

1. **Error: módulo no compilado antes de su uso**
```fortran
! Archivo: uso.f90
program uso
    use mi_modulo
    implicit none
    print *, hola
end program uso

! Archivo: modulo.f90
module mi_modulo
    implicit none
end module
```
Si se compila solo `gfortran uso.f90` sin compilar el módulo primero, el mensaje es: `Fatal Error: Cannot open module file 'mi_modulo.mod' for reading at (1): No such file or directory`. Solución: compilar en orden: `gfortran -c modulo.f90 && gfortran uso.f90 modulo.o`.

2. **Error: conflicto de nombres con `use`**
```fortran
program error_use
    implicit none
    integer :: PI = 3
    use operaciones_matematicas
    print *, PI
end program error_use
```
El `use` debe ir antes de `implicit none`. Además, si ya existe una variable local con el mismo nombre, hay conflicto. Mensaje: `Error: Symbol 'pi' at (1) already has basic type of INTEGER`. Solución: poner `use` primero o usar `use ... only: pi_mod => PI` (renombrar).

**Mini-proyecto: Módulo de utilidades matemáticas con estadísticas**

Un módulo completo que ofrece funciones para media, varianza, desviación estándar y correlación entre dos conjuntos de datos. El programa de prueba usa el módulo y demuestra todas las funciones.

```fortran
! modulo/estadisticas_mod.f90
module estadisticas_mod
    implicit none

    ! Constantes
    real, parameter :: TOLERANCIA = 1.0e-8

    ! Privado por defecto, público solo lo que se exporta
    private
    public :: media, varianza, desviacion_std, correlacion

contains

    ! Media aritmética
    function media(x) result(m)
        implicit none
        real, intent(in) :: x(:)
        real :: m
        m = sum(x) / real(size(x))
    end function media

    ! Varianza muestral
    function varianza(x) result(v)
        implicit none
        real, intent(in) :: x(:)
        real :: v, m
        integer :: n
        n = size(x)
        m = media(x)
        v = sum((x - m)**2) / real(n - 1)
    end function varianza

    ! Desviación estándar
    function desviacion_std(x) result(s)
        implicit none
        real, intent(in) :: x(:)
        real :: s
        s = sqrt(varianza(x))
    end function desviacion_std

    ! Coeficiente de correlación de Pearson entre x e y
    function correlacion(x, y) result(r)
        implicit none
        real, intent(in) :: x(:), y(:)
        real :: r, mx, my, cov, sx, sy
        integer :: n

        n = size(x)
        if (size(y) /= n) then
            write(*, *) 'Error: los arrays deben tener el mismo tamaño.'
            r = 0.0
            return
        end if

        mx = media(x)
        my = media(y)
        sx = desviacion_std(x)
        sy = desviacion_std(y)

        if (abs(sx) < TOLERANCIA .or. abs(sy) < TOLERANCIA) then
            write(*, *) 'Error: desviación estándar cero (datos constantes).'
            r = 0.0
            return
        end if

        cov = sum((x - mx) * (y - my)) / real(n - 1)
        r = cov / (sx * sy)
    end function correlacion

end module estadisticas_mod
```

```fortran
! modulo/prueba_estadisticas.f90
program prueba_estadisticas
    use estadisticas_mod
    implicit none

    real :: notas(5) = [85.0, 92.0, 78.0, 95.0, 88.0]
    real :: horas_estudio(5) = [4.0, 5.0, 2.0, 6.0, 4.0]
    integer :: i

    write(*, *) '=== MÓDULO DE ESTADÍSTICAS ==='
    write(*, *)
    write(*, *) 'Datos: Notas de estudiantes'
    do i = 1, 5
        write(*, *) '  Estudiante ', i, ': nota = ', notas(i), &
                     ', horas estudio = ', horas_estudio(i)
    end do

    write(*, *)
    write(*, '(A, F10.2)') ' Media de notas:             ', media(notas)
    write(*, '(A, F10.4)') ' Varianza de notas:          ', varianza(notas)
    write(*, '(A, F10.4)') ' Desviación estándar:        ', desviacion_std(notas)
    write(*, '(A, F10.4)') ' Correlación (notas vs estudio): ', correlacion(notas, horas_estudio)

end program prueba_estadisticas
```

**Análisis línea por línea (módulo)**:
- `private`: Cambia el acceso por defecto a privado. Sin esta línea, todo sería público. Con `private`, debemos marcar explícitamente qué queremos exportar.
- `public :: media, varianza, desviacion_std, correlacion`: Lista de símbolos públicos. Solo estos serán accesibles desde fuera. Si olvidamos añadir una función aquí, no será visible.
- `real, intent(in) :: x(:)`: Argumento de tipo array asumido (assumed-shape). El `(:)` indica un array unidimensional de cualquier tamaño. Esta notación solo funciona con interfaces explícitas (como las que proporcionan los módulos).
- `sum(x)`, `size(x)`: Funciones intrínsecas aplicadas al array completo. `sum(x)` suma todos los elementos. `size(x)` devuelve el número de elementos.
- `sum((x - m)**2) / real(n - 1)`: Varianza en una sola línea. `(x - m)` resta la media a cada elemento (operación vectorial). `**2` eleva al cuadrado cada elemento. `sum` suma todos.

**Salida esperada**:
```
 === MÓDULO DE ESTADÍSTICAS ===

 Datos: Notas de estudiantes
   Estudiante            1 : nota =    85.0000000 , horas estudio =    4.00000000
   Estudiante            2 : nota =    92.0000000 , horas estudio =    5.00000000
   Estudiante            3 : nota =    78.0000000 , horas estudio =    2.00000000
   Estudiante            4 : nota =    95.0000000 , horas estudio =    6.00000000
   Estudiante            5 : nota =    88.0000000 , horas estudio =    4.00000000

 Media de notas:                 87.60
 Varianza de notas:             42.8000
 Desviación estándar:            6.5422
 Correlación (notas vs estudio):  0.9499
```

---

## 7. I/O con formato

La entrada y salida con formato es el arte de controlar exactamente cómo se muestran los datos y cómo se leen desde archivos o desde la terminal. Sin formato, los números aparecen como el compilador decide: a veces con 6 decimales, a veces con notación científica. Con formato, podemos alinear columnas, controlar la precisión, añadir texto literal, y en general producir informes con aspecto profesional.

La instrucción `format` en Fortran define una plantilla para leer o escribir datos. Existen dos formas de usarla: formato explícito con `format` (una línea aparte) y formato empotrado en la instrucción `write` o `read` (entre paréntesis y comillas). La sintaxis del `format` usa códigos de edición: `I` para enteros, `F` para reales, `E` y `ES` para notación científica, `A` para caracteres, `X` para espacios, `/` para saltos de línea, y muchos más.

Pensemos en el formato como una plantilla de mecanografía: cada código de edición especifica qué tipo de dato esperar y cómo debe presentarse. Por ejemplo, `F10.4` significa "un número real (F), ocupando 10 caracteres en total, con 4 decimales". Si el número necesita más de 10 caracteres, Fortran imprime asteriscos (`********`) para indicar que el campo es demasiado pequeño. Esta señal de error es mucho más útil que la alternativa de C (que simplemente truncaría o desbordaría).

Las etiquetas `format` tradicionales son números (como `100 format(...)`), pero en Fortran moderno se recomienda usar cadenas literales empotradas en el `write`: `write(*, '(I4, F10.2)') i, x`. Esta forma es más legible porque el formato está junto a los datos que formatea. Sin embargo, para formatos muy largos o reutilizables, la etiqueta `format` sigue siendo útil.

El asterisco como formato (`write(*, *) ...`) es el formato "lista dirigida" (list-directed): el compilador elige automáticamente la presentación. Es cómodo para depuración, pero no sirve para producir informes profesionales porque no hay control sobre la alineación, precisión o formato de números muy grandes o muy pequeños.

Para leer datos con formato, `read` con formato puede extraer datos de posiciones específicas de una línea. Esto es crítico cuando se leen archivos de datos legacy donde los valores están en columnas fijas (formato de tarjeta perforada). Aunque esto es menos común hoy, sigue siendo necesario en muchos ámbitos científicos donde los datos históricos están en formatos de ancho fijo.

La entrada y salida de archivos en Fortran es conceptualmente simple: (1) abrir el archivo con `open`, (2) leer o escribir con `read`/`write` usando la unidad asignada, (3) cerrar con `close`. La unidad es un número entero que identifica el archivo (las unidades 5 y 6 son la entrada y salida estándar respectivamente). Podemos usar `newunit` (Fortran 2008+) para que el sistema asigne automáticamente una unidad libre.

### Formato en write y read

```fortran
program io_formateado
    implicit none

    integer :: i, n_alumnos
    real :: notas(5)
    character(len=30) :: nombres(5)
    character(len=20) :: archivo_salida
    integer :: unidad

    ! ============================================
    ! 1. FORMATO EXPLÍCITO CON ETIQUETA
    ! ============================================
    write(*, *) '=== I/O CON FORMATO ==='
    write(*, *)

    write(*, *) 'Formato con etiqueta FORMAT (100):'
    do i = 1, 5
        write(*, 100) i, i**2, i**3, sqrt(real(i))
    end do

    ! Etiqueta de formato
100 format(' i = ', I2, ' | i² = ', I3, ' | i³ = ', I4, ' | √i = ', F6.3)

    ! ============================================
    ! 2. FORMATO EMPOTRADO
    ! ============================================
    write(*, *)
    write(*, *) 'Formato empotrado (cadena literal):'

    ! Formato con códigos de edición
    write(*, '(A, I2, A)') 'El valor de i es: ', 42, ' (literal)'

    ! Tabla formateada con encabezados
    write(*, *)
    write(*, '(A)') 'Tabla de valores:'
    write(*, '(A)') '-----------------'
    write(*, '(A5, 2X, A10, 2X, A10)') 'Índice', 'Valor', 'Raíz'
    write(*, '(A5, 2X, A10, 2X, A10)') '------', '-----', '----'

    do i = 1, 5
        write(*, '(I5, 2X, F10.2, 2X, F10.4)') i, real(i * 10), sqrt(real(i))
    end do

    ! ============================================
    ! 3. NOTACIÓN CIENTÍFICA
    ! ============================================
    write(*, *)
    write(*, *) 'Notación científica (E y ES):'
    write(*, '(A10, 2X, A15, 2X, A15)') 'Valor', 'E (exponencial)', 'ES (ingenieril)'
    write(*, '(A10, 2X, A15, 2X, A15)') '-----', '----------------', '----------------'
    write(*, '(F10.2, 2X, E15.6, 2X, ES15.6)') 0.00123, 0.00123, 0.00123
    write(*, '(F10.2, 2X, E15.6, 2X, ES15.6)') 1234.56, 1234.56, 1234.56
    write(*, '(F10.2, 2X, E15.6, 2X, ES15.6)') -0.0000567, -0.0000567, -0.0000567

    ! ============================================
    ! 4. LECTURA CON FORMATO
    ! ============================================
    write(*, *)
    write(*, *) 'Lectura de datos con formato:'
    write(*, *) 'Ingrese nombre y nota de 2 alumnos (ej: "Juan 85.5"):'

    do i = 1, 2
        read(*, '(A, F6.2)') nombres(i), notas(i)
    end do

    write(*, *)
    write(*, '(A)') 'Datos ingresados:'
    write(*, '(A)') '-----------------'
    do i = 1, 2
        write(*, '(A, A, F6.2)') '  ', trim(nombres(i)), notas(i)
    end do

    ! ============================================
    ! 5. ESCRITURA A ARCHIVO
    ! ============================================
    write(*, *)
    write(*, *) 'Escritura a archivo...'

    archivo_salida = 'tabla_resultados.txt'

    ! Abrir archivo con NEWUNIT (Fortran 2008+)
    open(newunit=unidad, file=archivo_salida, status='replace', &
         action='write', iostat=i)
    if (i /= 0) then
        write(*, *) 'Error al abrir archivo.'
        stop
    end if

    ! Escribir encabezados y datos
    write(unidad, '(A)') '=== RESULTADOS GENERADOS ==='
    write(unidad, '(A)') '---------------------------'
    write(unidad, '(A5, 2X, A10, 2X, A10)') 'Índice', 'Valor', 'Raíz'

    do i = 1, 5
        write(unidad, '(I5, 2X, F10.2, 2X, F10.4)') i, real(i * 10), sqrt(real(i))
    end do

    ! Cerrar archivo
    close(unidad)
    write(*, *) 'Archivo "', trim(archivo_salida), '" creado exitosamente.'

    ! ============================================
    ! 6. LECTURA DESDE ARCHIVO
    ! ============================================
    write(*, *)
    write(*, *) 'Leyendo datos desde el archivo creado...'

    open(newunit=unidad, file=archivo_salida, status='old', action='read', iostat=i)
    if (i /= 0) then
        write(*, *) 'Error al leer archivo.'
        stop
    end if

    ! Leer y mostrar línea por línea
    n_alumnos = 0
    do
        read(unidad, '(A)', iostat=i) nombres(1)
        if (i /= 0) exit
        n_alumnos = n_alumnos + 1
        write(*, '(I3, A, A)') n_alumnos, ': ', trim(nombres(1))
    end do

    close(unidad)
    write(*, *) 'Total de líneas leídas: ', n_alumnos

end program io_formateado
```

**Análisis línea por línea**:
- `100 format(' i = ', I2, ' | i² = ', I3, ' | i³ = ', I4, ' | √i = ', F6.3)`: Etiqueta de formato. `I2` es un entero que ocupa hasta 2 posiciones. `F6.3` es un real con 6 posiciones totales y 3 decimales. Si el entero necesita más de 2 dígitos, se imprime `**`.
- `write(*, '(A, I2, A)') 'El valor de i es: ', 42, ' (literal)'`: Formato empotrado. `A` imprime una cadena. Los códigos se corresponden uno a uno con los argumentos.
- `'(I5, 2X, F10.2, 2X, F10.4)'`: `I5` (entero, 5 posiciones), `2X` (2 espacios), `F10.2` (real, 10 posiciones, 2 decimales), `2X`, `F10.4` (real, 10 posiciones, 4 decimales).
- `'(F10.2, 2X, E15.6, 2X, ES15.6)'`: `E15.6` es notación científica (un dígito antes del punto decimal). `ES15.6` es notación ingenieril (potencias múltiplo de 3).
- `read(*, '(A, F6.2)') nombres(i), notas(i)`: Lee un string (hasta 30 caracteres) y un real de hasta 6 posiciones con 2 decimales.
- `open(newunit=unidad, file=archivo_salida, status='replace', action='write', iostat=i)`: `newunit` asigna automáticamente una unidad libre. `status='replace'` sobreescribe el archivo si existe.
- `read(unidad, '(A)', iostat=i)`: Lee una línea completa como cadena. `iostat` es distinto de 0 cuando se llega al final del archivo (EOF). Sin `iostat`, un EOF causaría un error de runtime.
- `if (i /= 0) exit`: Sale del bucle cuando se alcanza el final del archivo. Sin esta condición, el bucle sería infinito.

**Salida esperada**:
```
 === I/O CON FORMATO ===

 Formato con etiqueta FORMAT (100):
 i =  1 | i² =   1 | i³ =    1 | √i =  1.000
 i =  2 | i² =   4 | i³ =    8 | √i =  1.414
 i =  3 | i² =   9 | i³ =   27 | √i =  1.732
 i =  4 | i² =  16 | i³ =   64 | √i =  2.000
 i =  5 | i² =  25 | i³ =  125 | √i =  2.236

 Formato empotrado (cadena literal):
 El valor de i es: 42 (literal)

 Tabla de valores:
 -----------------
 Índice  Valor       Raíz
 ------  -----       ----
     1      10.00      1.0000
     2      20.00      1.4142
     3      30.00      1.7321
     4      40.00      2.0000
     5      50.00      2.2361

 Notación científica (E y ES):
     Valor           E (exponencial)    ES (ingenieril)
     -----           ----------------    ----------------
       0.00         1.230000E-03        1.230000E-03
    1234.56         1.234560E+03        1.234560E+03
      -0.00        -5.670000E-05       -5.670000E-05

 Lectura de datos con formato:
 Ingrese nombre y nota de 2 alumnos (ej: "Juan 85.5"):
> Ana 92.3
> Luis 78.5

 Datos ingresados:
 -----------------
   Ana 92.30
   Luis 78.50

 Escritura a archivo...
 Archivo "tabla_resultados.txt" creado exitosamente.

 Leyendo datos desde el archivo creado...
  1: === RESULTADOS GENERADOS ===
  2: ---------------------------
  3: Índice  Valor       Raíz
  4:     1      10.00      1.0000
  5:     2      20.00      1.4142
  6:     3      30.00      1.7321
  7:     4      40.00      2.0000
  8:     5      50.00      2.2361
 Total de líneas leídas:            8
```

**Errores típicos**:

1. **Error: formato insuficiente para el valor**
```fortran
program error_formato
    implicit none
    integer :: x = 12345
    write(*, '(I3)') x  ! I3 solo admite hasta 3 dígitos
end program error_formato
```
Mensaje real: `Fortran runtime error: Positive width required in format at (1)` o simplemente imprime `***`. Solución: usar un descriptor suficientemente grande, como `I5`.

2. **Error: archivo abierto con `status='old'` que no existe**
```fortran
program error_open
    implicit none
    integer :: u
    open(newunit=u, file='no_existe.txt', status='old')
end program error_open
```
Mensaje real: `Fortran runtime error: Cannot open file 'no_existe.txt': No such file or directory`. Sin `iostat`, el programa aborta. Solución: verificar `iostat` o usar `status='new'`.

3. **Error: tipo de dato no coincide con formato**
```fortran
program error_tipo_formato
    implicit none
    real :: x = 3.14
    write(*, '(I5)') x  ! I espera integer, no real
end program error_tipo_formato
```
Mensaje real: `Fortran runtime error: Expected INTEGER for item 1 in formatted transfer, got REAL`. Solución: usar `F` o `E` para reales.

**Mini-proyecto: Generador de informes formateados**

Un programa que lee datos de ventas desde teclado y genera un informe formateado en pantalla y en un archivo de texto con alineación de columnas, totales y promedios.

```fortran
program generador_informe
    implicit none

    integer :: n, i, unidad, istat
    character(len=40), allocatable :: productos(:)
    real, allocatable :: ventas(:)
    real :: total, promedio, max_venta, min_venta
    integer :: idx_max, idx_min
    character(len=30) :: archivo = 'informe_ventas.txt'

    ! Leer cantidad de productos
    write(*, *) '=== GENERADOR DE INFORME DE VENTAS ==='
    write(*, *) '¿Cuántos productos desea registrar?'
    read(*, *) n

    if (n <= 0) then
        write(*, *) 'Error: debe registrar al menos un producto.'
        stop
    end if

    ! Asignar memoria dinámica
    allocate(productos(n), ventas(n), stat=istat)
    if (istat /= 0) then
        write(*, *) 'Error de memoria.'
        stop
    end if

    ! Leer datos
    write(*, *) 'Ingrese los datos (nombre y ventas) de cada producto:'
    do i = 1, n
        write(*, '(A, I2, A)') 'Producto ', i, ':'
        write(*, *) '  Nombre: '
        read(*, '(A)') productos(i)
        write(*, *) '  Ventas ($): '
        read(*, *) ventas(i)
    end do

    ! Calcular estadísticas
    total = sum(ventas)
    promedio = total / real(n)
    idx_max = maxloc(ventas, dim=1)
    idx_min = minloc(ventas, dim=1)
    max_venta = ventas(idx_max)
    min_venta = ventas(idx_min)

    ! Mostrar informe en pantalla
    write(*, *)
    write(*, '(A)') '==================================='
    write(*, '(A)') '       INFORME DE VENTAS'
    write(*, '(A)') '==================================='
    write(*, *)
    write(*, '(A, I3, A)') 'Productos registrados: ', n
    write(*, *)
    write(*, '(A)') 'Detalle:'
    write(*, '(A)') '--------'
    write(*, '(A5, 2X, A30, 2X, A12)') '#', 'Producto', 'Ventas ($)'
    write(*, '(A5, 2X, A30, 2X, A12)') '---', '-------', '----------'

    do i = 1, n
        if (i == idx_max) then
            write(*, '(I5, 2X, A30, 2X, F12.2, A)') i, trim(productos(i)), &
                                                     ventas(i), '  <<< MÁX'
        else if (i == idx_min) then
            write(*, '(I5, 2X, A30, 2X, F12.2, A)') i, trim(productos(i)), &
                                                     ventas(i), '  <<< MÍN'
        else
            write(*, '(I5, 2X, A30, 2X, F12.2)') i, trim(productos(i)), &
                                                  ventas(i)
        end if
    end do

    write(*, '(A5, 2X, A30, 2X, A12)') '---', '------------------------------', '-----------'
    write(*, '(A5, 2X, A30, 2X, F12.2)') '', 'TOTAL', total
    write(*, '(A5, 2X, A30, 2X, F12.2)') '', 'PROMEDIO', promedio
    write(*, '(A5, 2X, A30, 2X, F12.2)') '', 'MÁXIMO', max_venta
    write(*, '(A5, 2X, A30, 2X, F12.2)') '', 'MÍNIMO', min_venta
    write(*, *)

    ! Escribir informe a archivo
    open(newunit=unidad, file=archivo, status='replace', action='write', &
         iostat=istat)
    if (istat == 0) then
        write(unidad, '(A)') 'INFORME DE VENTAS GENERADO DESDE FORTRAN'
        write(unidad, '(A, I3)') 'Productos: ', n
        write(unidad, '(A)') ''
        write(unidad, '(A5, A30, A12)') '# ', 'Producto', ' Ventas($)'
        write(unidad, '(A5, A30, A12)') '--- ', '-------', ' ----------'

        do i = 1, n
            write(unidad, '(I5, A30, F12.2)') i, ' ' // trim(productos(i)), &
                                               ventas(i)
        end do

        write(unidad, '(A5, A30, A12)') '--- ', '-------', ' ----------'
        write(unidad, '(A5, A30, F12.2)') '', 'TOTAL', total
        write(unidad, '(A5, A30, F12.2)') '', 'PROMEDIO', promedio
        close(unidad)
        write(*, *) 'Informe guardado en: ', trim(archivo)
    else
        write(*, *) 'Error al crear el archivo de informe.'
    end if

    deallocate(productos, ventas)

end program generador_informe
```

**Análisis línea por línea**:
- `idx_max = maxloc(ventas, dim=1)`: `maxloc` devuelve el índice del elemento máximo. `dim=1` es obligatorio para arrays unidimensionales. Sin `dim=1`, `maxloc` devuelve un array de índice.
- `write(*, '(I5, 2X, A30, 2X, F12.2, A)') i, trim(productos(i)), ventas(i), '  <<< MÁX'`: Formato que alinea columnas y añade un marcador.
- `write(unidad, '(I5, A30, F12.2)') i, ' ' // trim(productos(i)), ventas(i)`: `//` es el operador de concatenación de cadenas.

**Salida esperada** (interactiva):
```
 === GENERADOR DE INFORME DE VENTAS ===
 ¿Cuántos productos desea registrar?
> 4
 Ingrese los datos (nombre y ventas) de cada producto:
 Producto  1:
   Nombre:
> Laptop
   Ventas ($):
> 15000.50
 Producto  2:
   Nombre:
> Monitor
   Ventas ($):
> 5200.00
 Producto  3:
   Nombre:
> Teclado
   Ventas ($):
> 850.75
 Producto  4:
   Nombre:
> Mouse
   Ventas ($):
> 320.00

 ===================================
       INFORME DE VENTAS
 ===================================

 Productos registrados:   4

 Detalle:
 --------
     #  Producto                          Ventas ($)
   ---  -------                           ----------
     1  Laptop                            15000.50  <<< MÁX
     2  Monitor                            5200.00
     3  Teclado                             850.75
     4  Mouse                               320.00  <<< MÍN
   ---  ------------------------------  -----------
         TOTAL                            21371.25
         PROMEDIO                          5342.81
         MÁXIMO                           15000.50
         MÍNIMO                             320.00

 Informe guardado en: informe_ventas.txt
```

---

## 8. Mini-proyecto: Calculadora Estadística Interactiva

Este mini-proyecto integra todos los conceptos del capítulo en una aplicación completa y funcional: una calculadora estadística interactiva. El programa permite al usuario cargar un conjunto de números, calcular estadísticas descriptivas y generar una tabla formateada con los resultados.

### Descripción del proyecto

La calculadora ofrece un menú interactivo con las siguientes opciones:
1. Ingresar datos (el usuario especifica cuántos números y los ingresa uno por uno)
2. Calcular estadísticas (media, varianza, desviación estándar, mínimo, máximo)
3. Mostrar datos originales en una tabla formateada
4. Salir

Todas las funciones estadísticas están encapsuladas en un módulo separado. Los datos se almacenan en un array dinámico (`allocatable`). El menú usa `select case`. La tabla de datos usa `format` para alineación. El programa principal es limpio y delega todo el trabajo a los procedimientos del módulo.

### Código completo

```fortran
! modulo/calculadora_estadistica_mod.f90
module calculadora_estadistica_mod
    implicit none

    ! Constantes del módulo
    real, parameter :: TOL = 1.0e-8

    ! Interfaz genérica para 'mostrar' (sobrecarga)
    interface mostrar
        module procedure mostrar_array_1d
    end interface mostrar

contains

    ! Subrutina para leer datos desde teclado
    subroutine leer_datos(arr, n)
        implicit none
        integer, intent(out) :: n
        real, allocatable, intent(out) :: arr(:)
        integer :: i, istat

        ! Leer tamaño
        write(*, *) '¿Cuántos números va a ingresar?'
        read(*, *) n

        if (n <= 0) then
            write(*, *) 'Error: la cantidad debe ser positiva.'
            n = 0
            return
        end if

        ! Asignar memoria dinámica
        allocate(arr(n), stat=istat)
        if (istat /= 0) then
            write(*, *) 'Error: no se pudo asignar memoria.'
            n = 0
            return
        end if

        ! Leer cada elemento
        write(*, *) 'Ingrese los ', n, ' números:'
        do i = 1, n
            write(*, '(A, I3, A)', advance='no') '  arr[', i, '] = '
            read(*, *) arr(i)
        end do

        write(*, *) 'Datos ingresados correctamente.'
    end subroutine leer_datos

    ! Función para calcular la media aritmética
    real function calcular_media(arr)
        implicit none
        real, intent(in) :: arr(:)
        calcular_media = sum(arr) / real(size(arr))
    end function calcular_media

    ! Función para calcular la varianza muestral (divide por n-1)
    real function calcular_varianza(arr)
        implicit none
        real, intent(in) :: arr(:)
        real :: media
        integer :: n
        n = size(arr)
        if (n <= 1) then
            calcular_varianza = 0.0
            return
        end if
        media = calcular_media(arr)
        calcular_varianza = sum((arr - media)**2) / real(n - 1)
    end function calcular_varianza

    ! Función para calcular la desviación estándar
    real function calcular_desviacion(arr)
        implicit none
        real, intent(in) :: arr(:)
        calcular_desviacion = sqrt(calcular_varianza(arr))
    end function calcular_desviacion

    ! Subrutina para encontrar mínimo y máximo
    subroutine encontrar_min_max(arr, minimo, maximo)
        implicit none
        real, intent(in) :: arr(:)
        real, intent(out) :: minimo, maximo
        minimo = minval(arr)
        maximo = maxval(arr)
    end subroutine encontrar_min_max

    ! Subrutina para mostrar datos en tabla formateada
    subroutine mostrar_array_1d(arr)
        implicit none
        real, intent(in) :: arr(:)
        integer :: i, n

        n = size(arr)
        write(*, *)
        write(*, '(A)') '=============================='
        write(*, '(A)') '       TABLA DE DATOS'
        write(*, '(A)') '=============================='
        write(*, '(A5, A)') '  #  ', '  Valor'
        write(*, '(A5, A)') ' --- ', '  -----'

        do i = 1, n
            write(*, '(I5, 2X, F12.4)') i, arr(i)
        end do

        write(*, '(A5, A)') ' --- ', '  -----'
        write(*, '(A5, I5)') '  N = ', n
        write(*, *)
    end subroutine mostrar_array_1d

    ! Subrutina para mostrar estadísticas en tabla formateada
    subroutine mostrar_estadisticas(arr)
        implicit none
        real, intent(in) :: arr(:)
        real :: media, varianza, desviacion, minimo, maximo
        integer :: n

        n = size(arr)
        if (n == 0) then
            write(*, *) 'Error: no hay datos cargados.'
            return
        end if

        ! Calcular todas las estadísticas
        media = calcular_media(arr)
        varianza = calcular_varianza(arr)
        desviacion = calcular_desviacion(arr)
        call encontrar_min_max(arr, minimo, maximo)

        ! Mostrar tabla de resultados
        write(*, *)
        write(*, '(A)') '=================================='
        write(*, '(A)') '     RESULTADOS ESTADÍSTICOS'
        write(*, '(A)') '=================================='
        write(*, *)
        write(*, '(A, A)') '  Estadística', '                  Valor'
        write(*, '(A, A)') '  ------------', '                  -----'

        write(*, '(A, F18.6)') '  Media aritmética           ', media
        write(*, '(A, F18.6)') '  Varianza muestral          ', varianza
        write(*, '(A, F18.6)') '  Desviación estándar        ', desviacion
        write(*, '(A, F18.6)') '  Valor mínimo               ', minimo
        write(*, '(A, F18.6)') '  Valor máximo               ', maximo
        write(*, *)
        write(*, '(A, I6)') '  Número de datos (N):       ', n
        write(*, *)

    end subroutine mostrar_estadisticas

end module calculadora_estadistica_mod
```

```fortran
! modulo/calculadora_estadistica.f90
program calculadora_estadistica
    use calculadora_estadistica_mod
    implicit none

    real, allocatable :: datos(:)
    integer :: n_datos, opcion
    logical :: datos_cargados = .false.

    ! Bucle principal del menú
    do
        ! Mostrar menú
        write(*, *)
        write(*, '(A)') '===================================='
        write(*, '(A)') '  CALCULADORA ESTADÍSTICA INTERACTIVA'
        write(*, '(A)') '===================================='
        write(*, '(A)') '  1. Ingresar datos'
        write(*, '(A)') '  2. Calcular estadísticas'
        write(*, '(A)') '  3. Mostrar datos'
        write(*, '(A)') '  4. Salir'
        write(*, '(A)') '------------------------------------'
        write(*, '(A)', advance='no') '  Opción: '
        read(*, *) opcion

        ! Procesar opción del menú
        select case (opcion)
            case (1)
                ! Ingresar nuevos datos
                if (allocated(datos)) then
                    deallocate(datos)
                end if
                call leer_datos(datos, n_datos)
                if (n_datos > 0) then
                    datos_cargados = .true.
                end if

            case (2)
                ! Calcular y mostrar estadísticas
                if (.not. datos_cargados) then
                    write(*, *) 'Error: primero debe ingresar datos (opción 1).'
                else
                    call mostrar_estadisticas(datos)
                end if

            case (3)
                ! Mostrar datos en tabla
                if (.not. datos_cargados) then
                    write(*, *) 'Error: primero debe ingresar datos (opción 1).'
                else
                    call mostrar(datos)
                end if

            case (4)
                ! Salir del programa
                write(*, *) '¡Gracias por usar la Calculadora Estadística!'
                if (allocated(datos)) then
                    deallocate(datos)
                end if
                exit

            case default
                write(*, *) 'Error: opción no válida (1-4).'
        end select

    end do

end program calculadora_estadistica
```

### Análisis línea por línea (módulo)

- `module calculadora_estadistica_mod`: Define el módulo que encapsula todas las funcionalidades de la calculadora. Debe compilarse antes que el programa principal.
- `real, parameter :: TOL = 1.0e-8`: Constante de tolerancia para comparaciones. Aunque no se usa explícitamente en este código, se incluye para demostrar la declaración de constantes y porque es buena práctica tener una tolerancia disponible.
- `interface mostrar`: Interfaz genérica que asocia el nombre `mostrar` con `mostrar_array_1d`. Aunque en esta versión solo tiene un procedimiento, la interfaz permite extenderla fácilmente para mostrar otros tipos de datos.
- `subroutine leer_datos(arr, n)`: Subrutina que lee datos desde teclado y los almacena en un array allocatable. `arr` es `intent(out)` porque se crea dentro de la subrutina.
- `allocate(arr(n), stat=istat)`: Asigna memoria para `n` elementos. `stat=istat` captura el código de error. Sin `stat`, un fallo de memoria detendría el programa abruptamente.
- `advance='no'` en `write(*, '(A, I3, A)', advance='no')`: Evita el avance de línea después del write, permitiendo que el `read` en la siguiente línea ocurra en la misma línea. Sin `advance='no'`, cada prompt aparecería en una línea separada del valor ingresado.
- `real function calcular_media(arr)`: Función pura (no tiene efectos secundarios). `arr(:)` es un array asumido (assumed-shape), que solo funciona con interfaces explícitas (como las que proveen los módulos).
- `sum(arr) / real(size(arr))`: Cálculo de la media en una sola línea usando funciones intrínsecas de Fortran. `size(arr)` devuelve el número de elementos. `real()` convierte a real para evitar división entera.
- `sum((arr - media)**2) / real(n - 1)`: Varianza muestral (divide por n-1). `(arr - media)` es una operación vectorial que resta la media a cada elemento. `**2` eleva cada elemento al cuadrado.
- `minval(arr) / maxval(arr)`: Funciones intrínsecas que devuelven el mínimo y máximo del array completo. Alternativa más compacta que un bucle manual.
- `call mostrar_array_1d(arr)`: Subrutina que muestra los datos en una tabla formateada. Usa formato `F12.4` para mostrar los números con 4 decimales alineados a la derecha.

### Análisis línea por línea (programa principal)

- `use calculadora_estadistica_mod`: Importa todos los símbolos públicos del módulo. El módulo debe estar compilado y accesible (archivo `.mod` generado).
- `real, allocatable :: datos(:)`: Array dinámico que almacenará los datos del usuario. Inicialmente no está asignado.
- `logical :: datos_cargados = .false.`: Indicador que controla si hay datos disponibles para procesar. Sin este indicador, el usuario podría calcular estadísticas sin haber ingresado datos, lo que causaría un error de runtime.
- `do`: Bucle infinito del menú. Termina solo cuando el usuario selecciona la opción 4, que ejecuta `exit`.
- `write(*, '(A)', advance='no') '  Opción: '`: Muestra el prompt sin salto de línea. `advance='no'` mantiene el cursor en la misma línea para que la entrada del usuario aparezca después de los dos puntos.
- `select case (opcion)`: Estructura de selección múltiple que dirige el flujo según la opción elegida. Solo funciona con tipos enteros, caracteres o lógicos.
- `if (allocated(datos)) then deallocate(datos)`: Libera memoria si había datos previos antes de cargar nuevos. Sin esta verificación, si el usuario ingresa datos por segunda vez sin haber liberado la memoria antes, `allocate` podría fallar o crear una fuga de memoria.
- `if (n_datos > 0) then datos_cargados = .true.`: Solo marca datos como cargados si la lectura fue exitosa. Si `n_datos == 0` (porque la entrada era inválida), el flag permanece falso.
- `if (.not. datos_cargados) then write(*, *) 'Error: primero debe ingresar datos (opción 1).'`: Verificación de precondición. Sin esta verificación, llamar a `mostrar_estadisticas(datos)` con un array no asignado causaría un error de runtime.
- `call mostrar(datos)`: Invoca la interfaz genérica `mostrar`, que se resuelve en `mostrar_array_1d`. El compilador selecciona el procedimiento correcto basado en el tipo y número de argumentos.
- `exit`: Termina el bucle infinito del menú. Sin `exit`, el programa nunca terminaría (excepto por interrupción del usuario con Ctrl+C).
- `deallocate(datos)`: Libera la memoria antes de salir. En este caso, como el programa termina, el sistema operativo recupera la memoria automáticamente, pero es buena práctica liberarla explícitamente.

### Salida esperada

```
====================================
  CALCULADORA ESTADÍSTICA INTERACTIVA
====================================
  1. Ingresar datos
  2. Calcular estadísticas
  3. Mostrar datos
  4. Salir
------------------------------------
  Opción: 1
 ¿Cuántos números va a ingresar?
> 6
 Ingrese los 6 números:
  arr[  1] = 12.5
  arr[  2] = 15.3
  arr[  3] = 9.8
  arr[  4] = 22.1
  arr[  5] = 18.6
  arr[  6] = 14.7
 Datos ingresados correctamente.

====================================
  CALCULADORA ESTADÍSTICA INTERACTIVA
====================================
  1. Ingresar datos
  2. Calcular estadísticas
  3. Mostrar datos
  4. Salir
------------------------------------
  Opción: 3

==============================
       TABLA DE DATOS
==============================
  #     Valor
 ---    -----
    1      12.5000
    2      15.3000
    3       9.8000
    4      22.1000
    5      18.6000
    6      14.7000
 ---    -----
  N =     6

====================================
  CALCULADORA ESTADÍSTICA INTERACTIVA
====================================
  1. Ingresar datos
  2. Calcular estadísticas
  3. Mostrar datos
  4. Salir
------------------------------------
  Opción: 2

==================================
     RESULTADOS ESTADÍSTICOS
==================================

  Estadística                  Valor
  ------------                  -----
  Media aritmética            15.500000
  Varianza muestral           17.636002
  Desviación estándar          4.199524
  Valor mínimo                 9.800000
  Valor máximo                22.100000

  Número de datos (N):             6

====================================
  CALCULADORA ESTADÍSTICA INTERACTIVA
====================================
  1. Ingresar datos
  2. Calcular estadísticas
  3. Mostrar datos
  4. Salir
------------------------------------
  Opción: 4
 ¡Gracias por usar la Calculadora Estadística!
```

---

## Interfaces explícitas e implícitas

Fortran tiene un concepto que, aunque parece técnico, es fundamental para escribir código correcto y moderno: las interfaces. Una interfaz es la declaración de cómo se debe llamar a un procedimiento: qué argumentos espera, de qué tipo son, y qué devuelve. Las interfaces pueden ser explícitas (el compilador conoce la firma completa) o implícitas (el compilador asume basándose en convenciones antiguas).

Cuando usamos módulos, las interfaces son automáticamente explícitas. El compilador puede verificar en tiempo de compilación que estamos pasando los argumentos correctos. Sin módulos, si llamamos a una función externa sin proporcionar su interfaz, el compilador asume reglas implícitas: que todos los argumentos son pasados por referencia y que el tipo de retorno sigue las reglas de tipado implícito.

Las interfaces explícitas permiten:
- Verificación de tipos y número de argumentos en compilación
- Paso de arrays asumidos (`arr(:)`)
- Argumentos con `intent` verificados
- Argumentos opcionales (`optional`)
- Sobrecarga de procedimientos (generic interfaces)

Las interfaces implícitas (propias de Fortran 77 y anteriores) pueden ocultar errores graves: pasar un entero donde se espera un real, pasar 3 argumentos donde se esperan 4, o asumir que una función devuelve entero cuando en realidad devuelve real.

La regla de oro: **siempre usa interfaces explícitas**. La forma más sencilla de lograrlo es poner todo en módulos. Para código legacy o cuando no es posible usar módulos, se pueden escribir bloques `interface` manualmente.

```fortran
! programa/interfaces.f90
module mis_procedimientos
    implicit none
contains

    ! Función con interfaz explícita (automática por el módulo)
    real function duplicar(x)
        implicit none
        real, intent(in) :: x
        duplicar = 2.0 * x
    end function duplicar

    ! Subrutina con interfaz explícita
    subroutine triplicar(x, resultado)
        implicit none
        real, intent(in) :: x
        real, intent(out) :: resultado
        resultado = 3.0 * x
    end subroutine triplicar

end module mis_procedimientos

program ejemplo_interfaces
    use mis_procedimientos
    implicit none

    real :: a = 5.0, b

    ! Llamadas verificadas por el compilador
    b = duplicar(a)
    write(*, *) 'Duplicado: ', b

    call triplicar(a, b)
    write(*, *) 'Triplicado: ', b

end program ejemplo_interfaces
```

**Análisis línea por línea**:
- `use mis_procedimientos`: Al usar el módulo, obtenemos interfaces explícitas automáticas. El compilador sabe que `duplicar` espera un `real` y devuelve un `real`.
- `b = duplicar(a)`: El compilador verifica que `a` es `real` (correcto) y que `duplicar` devuelve `real` (coincide con `b`). Si pasáramos un entero, el compilador nos advertiría.
- `call triplicar(a, b)`: El compilador verifica que `a` es `intent(in)` y `b` es `intent(out)`. Si intercambiáramos los argumentos, el compilador detectaría la incompatibilidad.

**Salida esperada**:
```
 Duplicado:    10.0000000
 Triplicado:    15.0000000
```

**Errores típicos**:

1. **Error: llamada a procedimiento externo sin interfaz explícita**
```fortran
! archivo: externo.f90
function cuadrado(x)
    implicit none
    real, intent(in) :: x
    real :: cuadrado
    cuadrado = x * x
end function

! archivo: main.f90
program main
    implicit none
    real :: a = 3.0
    ! Sin interfaz explícita: el compilador asume reglas implícitas
    write(*, *) cuadrado(a)
end program main
```
Si se compila y enlaza: `gfortran main.f90 externo.f90`, puede no dar error de compilación pero el resultado podría ser incorrecto si las convenciones de llamada no coinciden. Solución: usar módulos o declarar `interface` en el programa principal.

---

## Conclusión

Hemos recorrido un camino que va desde la instalación del compilador hasta la construcción de una calculadora estadística interactiva completa. En este viaje, hemos explorado los fundamentos del Fortran moderno: tipos de datos, arrays estáticos y dinámicos, control de flujo, funciones y subrutinas, módulos, y entrada/salida con formato.

Fortran es un lenguaje que ha sabido evolucionar sin perder su esencia. A diferencia de otros lenguajes que cambian radicalmente entre versiones, Fortran ha mantenido una compatibilidad hacia atrás asombrosa: programas escritos en Fortran 77 pueden compilarse con un compilador moderno. Pero al mismo tiempo, el Fortran moderno ofrece construcciones que lo ponen a la altura de cualquier lenguaje contemporáneo: tipos derivados, programación orientada a objetos, genéricos, y paralelismo nativo.

Lo más importante que debes recordar de este capítulo es que la claridad y la corrección son más importantes que la brevedad. Usa `implicit none` siempre. Declara `intent` en todos los argumentos. Usa módulos para organizar el código. Prefiere `allocatable` sobre `pointer`. Verifica los errores de `allocate` y `open`. Y sobre todo, escribe código que otro ser humano (incluso tú mismo dentro de seis meses) pueda entender sin esfuerzo.

El camino para dominar Fortran no termina aquí. El siguiente paso natural es explorar los tipos derivados (derived types), que permiten crear estructuras de datos complejas combinando diferentes tipos. Luego, la programación orientada a objetos con Fortran 2003+, que añade herencia, polimorfismo y encapsulamiento real. Después, el manejo de archivos binarios, la interfaz con C (ISO_C_BINDING), y finalmente la computación paralela con OpenMP, MPI y coarrays.

Pero todo eso se construye sobre los cimientos que hemos establecido aquí. Cada concepto que has aprendido —desde el tipado explícito hasta los módulos con alcance público/privado— es un ladrillo en ese edificio. La calculadora estadística que construimos no es solo un ejercicio: es un ejemplo del tipo de aplicaciones que puedes crear con Fortran moderno: robustas, eficientes, bien organizadas y profesionales.

La mejor forma de aprender es practicar. Toma la calculadora estadística y extiéndela: añade mediana, moda, cuartiles. Agrega la opción de leer datos desde un archivo. Implementa una regresión lineal simple. Cada mejora que hagas consolidará lo aprendido y te revelará nuevos aspectos del lenguaje.
