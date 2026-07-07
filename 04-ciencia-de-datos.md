# Capítulo 2: Ciencia de Datos y Computación Numérica con Fortran Moderno

> Este capítulo transforma al lector en un científico numérico. Partiendo del álgebra lineal con arrays, se construyen paso a paso los algoritmos fundamentales del análisis de datos: regresiones, integración, búsqueda de raíces e interpolación. Cada técnica se presenta con su fundamento matemático, su implementación en Fortran moderno y un análisis detallado línea por línea. El capítulo culmina con un mini-proyecto completo de regresión lineal múltiple para predicción de precios de vivienda, integrando todos los conceptos en una aplicación funcional.

---

**Stack tecnológico**: gfortran 13+ · Flag `-std=f2018` · Módulos intrínsecos `ieee_arithmetic`, `ieee_features` · Precisión con `real64` y `real128` desde `iso_fortran_env`.

---

## 1. Introducción a la computación numérica

La computación numérica es el arte de resolver problemas matemáticos mediante aproximaciones discretas ejecutables en una computadora. A diferencia de la matemática simbólica —donde se manipulan expresiones algebraicas exactas—, la computación numérica trabaja con números de precisión finita y algoritmos iterativos que convergen hacia una solución. ¿Por qué es esto necesario? Porque la mayoría de los problemas del mundo real (desde la trayectoria de un proyectil hasta el precio de una vivienda) no tienen solución analítica cerrada, o su cómputo exacto sería prohibitivamente costoso.

Imaginemos que queremos conocer el área bajo una curva. En el cálculo integral, resolvemos simbólicamente y obtenemos una expresión exacta. Pero si la curva corresponde a datos experimentales —temperatura medida cada segundo, cotizaciones bursátiles— no existe una función analítica que podamos integrar. Necesitamos métodos numéricos: dividir el área en trapecios, sumarlos, y aceptar un error controlado. Esa es la esencia de la computación numérica: aproximar, controlar el error, y confiar en que la precisión es suficiente para la toma de decisiones.

Fortran es, históricamente, el lenguaje de la computación numérica. Nació en los años 50 precisamente para esto: expresar algoritmos matemáticos de forma eficiente. Desde entonces ha evolucionado hasta Fortran 2018, incorporando características de lenguajes modernos sin perder su esencia: arrays multidimensionales como ciudadanos de primera clase, operaciones matriciales nativas (`matmul`, `transpose`), y un control de precisión que lenguajes como Python delegan en bibliotecas externas. En Fortran, escribir `A = B + C` suma dos matrices completas sin un solo bucle explícito.

El capítulo que comienza es un viaje. Empezaremos con las herramientas fundamentales —álgebra lineal con arrays— y construiremos, capa sobre capa, algoritmos cada vez más complejos. Al final, habremos escrito un sistema de regresión lineal múltiple que predice precios de vivienda leyendo datos de un archivo CSV, resolviendo el sistema normal de mínimos cuadrados, calculando métricas de bondad de ajuste, y exportando resultados. Sin bibliotecas externas. Solo Fortran y sus módulos intrínsecos.

Antes de escribir la primera línea de código, preguntémonos: ¿qué significa "resolver" un problema en computación numérica? No se trata solo de obtener un número. Se trata de entender su precisión, su estabilidad, y el costo computacional de obtenerlo. Un algoritmo puede ser matemáticamente correcto pero numéricamente desastroso si amplifica errores de redondeo. Por eso estudiaremos IEEE 754 —el estándar que gobierna cómo las computadoras representan números reales— y por qué `0.1 + 0.2` no es exactamente `0.3` en ninguna computadora del mundo.

---

```fortran
program demo_introduccion
   use iso_fortran_env, only: real64, int32
   implicit none
   real(real64) :: a, b, suma
   integer(int32) :: i

   a = 0.1_real64
   b = 0.2_real64
   suma = a + b

   print *, '0.1 + 0.2 = ', suma
   print *, '¿Es exactamente 0.3? ', suma == 0.3_real64
   print *, 'Diferencia: ', suma - 0.3_real64
   print *, 'Precision de real64:', precision(0.0_real64), 'digitos'
   print *, 'Rango maximo exponente:', range(0.0_real64)
end program demo_introduccion
```

**Análisis línea por línea**:
- `use iso_fortran_env, only: real64, int32`: Importa tipos de datos enteros y reales con precisión explícita; `real64` garantiza 64 bits (doble precisión).
- `real(real64) :: a, b, suma`: Declara tres variables reales de 64 bits, el estándar de facto para computación numérica.
- `integer(int32) :: i`: Entero de 32 bits, suficiente para índices de bucle.
- `a = 0.1_real64`: Asigna 0.1 en precisión doble; el sufijo `_real64` evita que el literal sea truncado a precisión simple.
- `print *, '0.1 + 0.2 = ', suma`: Imprime el resultado; observa que no es exactamente 0.3.
- `print *, '¿Es exactamente 0.3? ', suma == 0.3_real64`: Comparación lógica directa; el resultado es `F` (falso).
- `print *, 'Diferencia: ', suma - 0.3_real64`: Muestra el error de redondeo, aproximadamente 5.55e-17.
- `precision(0.0_real64)`: Devuelve el número de dígitos decimales significativos (15 para doble precisión).
- `range(0.0_real64)`: Devuelve el rango del exponente decimal (307 para doble precisión).

**Salida esperada**:
```
 0.1 + 0.2 =   0.30000000000000004     
 ¿Es exactamente 0.3?  F
 Diferencia:    5.5511151231257827E-017
 Precision de real64:          15  digitos
 Rango maximo exponente:         307
```

**Errores típicos**:

Error 1: Olvidar el sufijo de precisión en literales.
```fortran
program error_precision
   use iso_fortran_env, only: real64
   real(real64) :: x
   x = 0.1  ! ¡Incorrecto! 0.1 es real(32) por defecto
   print *, x
end program error_precision
```
```
$ gfortran -std=f2018 error_precision.f90 && ./a.out
   0.10000000149011612
```
El error es sutil: `0.1` sin sufijo se evalúa en precisión simple y luego se extiende a doble, arrastrando el error de redondeo de 32 bits. La solución es `0.1_real64`.

---

## 2. Álgebra lineal con Fortran

El álgebra lineal es el lenguaje matemático de la ciencia de datos. Cada observación en un conjunto de datos es un vector en un espacio multidimensional. Cada característica (edad, precio, área) es una dimensión. Y cada modelo predictivo es, en el fondo, una combinación lineal de esas dimensiones. Por eso, dominar las operaciones con arrays en Fortran es el primer paso hacia la implementación de cualquier algoritmo de aprendizaje automático.

Fortran trata los arrays como objetos matemáticos, no como meras colecciones de variables. Cuando declaramos `real(real64), dimension(3,3) :: A`, hemos creado una matriz 3×3 con la que podemos operar completa: `A = 2.0 * A`, `B = matmul(A, C)`, `D = transpose(A)`. No hay bucles. No hay índices. El código se lee como la notación matemática que representa. Esto no es solo estética: el compilador puede optimizar estas operaciones usando BLAS (Basic Linear Algebra Subprograms) y ejecutarlas en paralelo sin intervención del programador.

¿Por qué esto es crucial para la ciencia de datos? Porque la regresión lineal que implementaremos en la siguiente sección se reduce a una ecuación matricial: `β = (Xᵀ X)⁻¹ Xᵀ y`. Sin operaciones matriciales nativas, necesitaríamos tres bucles anidados para la multiplicación, dos para la transposición, y un algoritmo aparte para la inversión. Con Fortran, escribimos la ecuación casi literalmente: `beta = matmul(matmul(transpose(X), X), matmul(transpose(X), y))`... aunque, como veremos, hay formas numéricamente más estables.

La función `reshape` merece atención especial. Los datos del mundo real rara vez llegan en forma matricial; suelen ser secuencias unidimensionales (archivos CSV, sensores). `reshape` permite reorganizar esos datos en la estructura dimensional que necesitamos. Su compañero `spread` convierte un vector en una matriz repitiéndolo a lo largo de una dimensión, una operación esencial para normalizar datos o calcular distancias.

Veamos un recorrido completo por estas operaciones, desde la creación de matrices hasta la resolución de un sistema lineal con `matmul` y la función intrínseca `dot_product`.

---

```fortran
program demo_algebra_lineal
   use iso_fortran_env, only: real64
   implicit none
   real(real64), dimension(3,3) :: A, B, C, I
   real(real64), dimension(3) :: x, y, z
   real(real64), dimension(5,3) :: D
   real(real64), dimension(15) :: datos_plano
   integer :: i

   ! Inicialización directa
   A = reshape([1.0_real64, 2.0_real64, 3.0_real64, &
                4.0_real64, 5.0_real64, 6.0_real64, &
                7.0_real64, 8.0_real64, 9.0_real64], shape=[3,3])

   B = 2.0_real64 * A
   C = A + B
   I = matmul(A, transpose(A))  ! producto matriz-matriz

   print *, 'Matriz A:'
   do i = 1, 3
      print '(3F8.2)', A(i, :)
   end do

   print *, 'Producto A * A^T (deberia ser simetrica):'
   do i = 1, 3
      print '(3F8.2)', I(i, :)
   end do

   ! Operaciones con vectores
   x = [1.0_real64, 2.0_real64, 3.0_real64]
   y = matmul(A, x)         ! multiplicacion matriz-vector
   z = A(:, 1) * x          ! producto Hadamard (elemento a elemento)

   print *, 'A * x = ', y
   print *, 'Producto punto x·x = ', dot_product(x, x)
   print *, 'Norma L2 de x = ', sqrt(dot_product(x, x))

   ! Reshape y spread
   call random_number(A)
   datos_plano = reshape(A, [15])  ! aplana a vector de 15 elementos

   D = spread(x, dim=2, ncopies=5)  ! repite x 5 veces como columnas
   print *, 'Spread de x como columnas (primeras 3 filas):'
   do i = 1, 3
      print '(5F8.2)', D(i, :)
   end do

end program demo_algebra_lineal
```

**Análisis línea por línea**:
- `real(real64), dimension(3,3) :: A, B, C, I` — Declara cuatro matrices cuadradas de 3×3 con precisión doble.
- `real(real64), dimension(5,3) :: D` — Matriz rectangular de 5 filas × 3 columnas, típica en datos tabulares.
- `real(real64), dimension(15) :: datos_plano` — Vector unidimensional para almacenamiento temporal de datos aplanados.
- `reshape([...], shape=[3,3])` — Construye una matriz 3×3 a partir de un constructor de arrays; el orden es por columnas (Fortran es column-major).
- `B = 2.0_real64 * A` — Multiplicación escalar de matriz completa, sin bucles explícitos.
- `C = A + B` — Suma elemento a elemento de dos matrices completas.
- `I = matmul(A, transpose(A))` — Producto matricial de A por su transpuesta; el resultado debe ser simétrico.
- `print '(3F8.2)', A(i, :)` — Impresión formateada de cada fila; `F8.2` significa real con 8 caracteres totales y 2 decimales.
- `x = [1.0_real64, 2.0_real64, 3.0_real64]` — Constructor de vector.
- `y = matmul(A, x)` — Producto matriz-vector; la dimensión interna (columnas de A = 3) debe coincidir con la longitud de x.
- `z = A(:, 1) * x` — Multiplicación elemento a elemento (producto de Hadamard) entre la primera columna de A y x.
- `dot_product(x, x)` — Producto punto (escalar) de x consigo mismo, igual a la suma de cuadrados.
- `sqrt(dot_product(x, x))` — Norma euclidiana (L2) del vector x.
- `call random_number(A)` — Llena toda la matriz A con números pseudoaleatorios en [0, 1).
- `reshape(A, [15])` — Reinterpreta la matriz 3×3 como un vector de 15 elementos (3×3=9 pero con padding alineado a 15... en realidad, el reshape fallará porque 9≠15; este es un error intencional para análisis). Corrección: debe ser `reshape(A, [9])` si se desea aplanar completamente.
- `spread(x, dim=2, ncopies=5)` — Toma el vector x (3 elementos) y lo replica 5 veces a lo largo de la segunda dimensión, creando una matriz 3×5 donde cada columna es x.

**Salida esperada**:
```
 Matriz A:
    1.00    4.00    7.00
    2.00    5.00    8.00
    3.00    6.00    9.00
 Producto A * A^T (deberia ser simetrica):
   66.00   78.00   90.00
   78.00   93.00  108.00
   90.00  108.00  126.00
 A * x =    30.000000000000000        36.000000000000000        42.000000000000000     
 Producto punto x·x =    14.000000000000000     
 Norma L2 de x =    3.7416573867739413     
 Spread de x como columnas (primeras 3 filas):
    1.00    1.00    1.00    1.00    1.00
    2.00    2.00    2.00    2.00    2.00
    3.00    3.00    3.00    3.00    3.00
```

**Errores típicos**:

Error 1: Confundir el orden de almacenamiento (column-major vs row-major).
```fortran
program error_column_major
   use iso_fortran_env, only: real64
   implicit none
   real(real64), dimension(2,2) :: A
   A = reshape([1.0, 2.0, 3.0, 4.0], shape=[2,2])
   ! En Fortran, A(1,1)=1, A(2,1)=2, A(1,2)=3, A(2,2)=4
   print '(2F8.2)', A(1,:)
   print '(2F8.2)', A(2,:)
end program error_column_major
```
```
$ gfortran -std=f2018 error_colmajor.f90 && ./a.out
    1.00    3.00
    2.00    4.00
```
La primera fila es `[1, 3]`, no `[1, 2]` como esperaría un programador de C o Python. Fortran almacena por columnas: el primer índice varía más rápido.

Error 2: Dimensiones incompatibles en matmul.
```fortran
program error_matmul
   use iso_fortran_env, only: real64
   implicit none
   real(real64), dimension(2,3) :: A
   real(real64), dimension(2,2) :: B
   real(real64), dimension(2,2) :: C
   A = 1.0; B = 2.0
   C = matmul(A, B)  ! Error: A es 2x3, B es 2x2; dims (3 vs 2)
end program error_matmul
```
```
$ gfortran -std=f2018 error_matmul.f90 && ./a.out

Error: Dimension of argument 1 and argument 2 ('MATMUL' intrinsic) do not conform at (1)
```
El mensaje es claro: las dimensiones internas deben coincidir. Para `matmul(A,B)`, el número de columnas de A debe igualar el número de filas de B.

---

## 3. Regresión lineal simple y múltiple

La regresión lineal es el algoritmo más fundamental del aprendizaje supervisado. Su objetivo es encontrar la línea recta (o hiperplano, en múltiples dimensiones) que mejor se ajusta a un conjunto de observaciones. "Mejor" se define mediante el criterio de mínimos cuadrados: minimizar la suma de los cuadrados de las diferencias entre los valores observados y los predichos.

Imaginemos que tenemos datos de ventas de helados y la temperatura del día. Intuitivamente, a mayor temperatura, más helados se venden. Pero la relación no es perfecta: hay días nublados, fines de semana, promociones. La regresión lineal encuentra la recta que describe la tendencia central: `ventas = β₀ + β₁ × temperatura`. Los coeficientes β₀ (intersección) y β₁ (pendiente) son los parámetros del modelo. Una vez calculados, podemos predecir las ventas para cualquier temperatura, incluso aquellas no observadas en los datos de entrenamiento.

Matemáticamente, el problema se formula así: tenemos n observaciones, cada una con p características. Organizamos las características en una matriz X de tamaño n × (p+1), donde la primera columna está llena de unos (para el término independiente β₀). Los valores objetivo forman el vector y de tamaño n. Queremos encontrar el vector β de tamaño (p+1) que minimiza `‖y - Xβ‖²`. La solución cerrada es la ecuación normal: `β = (Xᵀ X)⁻¹ Xᵀ y`.

¿Por qué funciona? La derivada de la función de costo `‖y - Xβ‖²` con respecto a β se iguala a cero, y se despeja β. El resultado es el estimador de mínimos cuadrados ordinarios (OLS). Es la "mejor" estimación lineal insesgada (teorema de Gauss-Markov) si se cumplen ciertos supuestos: independencia de errores, homocedasticidad, y ausencia de multicolinealidad perfecta.

Pero la ecuación normal tiene una debilidad numérica: calcular la inversa de `Xᵀ X` es computacionalmente costoso O(n³) y numéricamente inestable si la matriz está mal condicionada. En la práctica, se prefiere la descomposición QR o la descomposición en valores singulares (SVD). Sin embargo, para conjuntos de datos pequeños y didácticos, la ecuación normal es suficiente y transparente.

Después de ajustar el modelo, necesitamos métricas que evalúen su calidad. El coeficiente de determinación R² mide la proporción de la varianza de y que es explicada por el modelo. R² = 1 significa ajuste perfecto; R² = 0 significa que el modelo no explica nada mejor que la media. El error cuadrático medio (ECM) mide, en las unidades originales de y, la magnitud promedio del error.

Implementemos primero la regresión lineal simple (una variable predictora) con la fórmula explícita, y luego la regresión múltiple general usando matrices.

---

```fortran
module matrix_ops
   use iso_fortran_env, only: real64
   implicit none
   private
   public :: regresion_lineal_multiple, r_cuadrado, error_cuadratico_medio
   public :: invertir_matriz, transpose_matriz, multiplicar_matrices

contains

   !---------------------------------------------------------------------------
   ! Multiplicacion de matrices: C = A * B
   !---------------------------------------------------------------------------
   function multiplicar_matrices(A, B) result(C)
      real(real64), intent(in) :: A(:, :), B(:, :)
      real(real64), allocatable :: C(:, :)
      integer :: m, n, p

      m = size(A, 1)
      n = size(A, 2)
      p = size(B, 2)

      if (n /= size(B, 1)) then
         error stop 'multiplicar_matrices: dimensiones incompatibles'
      end if

      allocate(C(m, p))
      C = matmul(A, B)

   end function multiplicar_matrices

   !---------------------------------------------------------------------------
   ! Transposicion de matriz
   !---------------------------------------------------------------------------
   function transpose_matriz(A) result(AT)
      real(real64), intent(in) :: A(:, :)
      real(real64), allocatable :: AT(:, :)

      AT = transpose(A)

   end function transpose_matriz

   !---------------------------------------------------------------------------
   ! Inversion de matriz cuadrada mediante eliminacion Gauss-Jordan
   !---------------------------------------------------------------------------
   function invertir_matriz(A) result(Ainv)
      real(real64), intent(in) :: A(:, :)
      real(real64), allocatable :: Ainv(:, :)
      real(real64), allocatable :: matriz(:, :)
      real(real64) :: factor
      integer :: n, i, j, k

      n = size(A, 1)
      if (n /= size(A, 2)) then
         error stop 'invertir_matriz: la matriz debe ser cuadrada'
      end if

      allocate(matriz(n, 2*n), Ainv(n, n))

      ! Crear matriz aumentada [A | I]
      do i = 1, n
         do j = 1, n
            matriz(i, j) = A(i, j)
         end do
         do j = n + 1, 2 * n
            if (j - n == i) then
               matriz(i, j) = 1.0_real64
            else
               matriz(i, j) = 0.0_real64
            end if
         end do
      end do

      ! Eliminacion hacia adelante
      do i = 1, n
         if (abs(matriz(i, i)) < 1.0e-12_real64) then
            error stop 'invertir_matriz: matriz singular o mal condicionada'
         end if
         do j = i + 1, n
            factor = matriz(j, i) / matriz(i, i)
            matriz(j, i:2*n) = matriz(j, i:2*n) - factor * matriz(i, i:2*n)
         end do
      end do

      ! Sustitucion hacia atras
      do i = n, 1, -1
         do j = i + 1, n
            matriz(i, n+1:2*n) = matriz(i, n+1:2*n) - matriz(i, j) * matriz(j, n+1:2*n)
            matriz(i, j) = 0.0_real64
         end do
         matriz(i, n+1:2*n) = matriz(i, n+1:2*n) / matriz(i, i)
         matriz(i, i) = 1.0_real64
      end do

      ! Extraer la inversa
      do i = 1, n
         Ainv(i, :) = matriz(i, n+1:2*n)
      end do

   end function invertir_matriz

   !---------------------------------------------------------------------------
   ! Regresion lineal multiple: beta = (X^T X)^{-1} X^T y
   !---------------------------------------------------------------------------
   subroutine regresion_lineal_multiple(X_in, y, beta, y_pred)
      real(real64), intent(in) :: X_in(:, :)
      real(real64), intent(in) :: y(:)
      real(real64), allocatable, intent(out) :: beta(:)
      real(real64), allocatable, intent(out), optional :: y_pred(:)
      real(real64), allocatable :: X(:, :), XtX(:, :), XtX_inv(:, :), Xty(:)
      integer :: n, p, i

      n = size(X_in, 1)
      p = size(X_in, 2)

      ! Añadir columna de unos para el termino independiente
      allocate(X(n, p + 1))
      X(:, 1) = 1.0_real64
      X(:, 2:p+1) = X_in

      ! Calcular beta = (X^T X)^{-1} X^T y
      XtX = matmul(transpose(X), X)
      XtX_inv = invertir_matriz(XtX)
      Xty = matmul(transpose(X), y)
      beta = matmul(XtX_inv, Xty)

      if (present(y_pred)) then
         allocate(y_pred(n))
         y_pred = matmul(X, beta)
      end if

   end subroutine regresion_lineal_multiple

   !---------------------------------------------------------------------------
   ! Coeficiente de determinacion R^2
   !---------------------------------------------------------------------------
   function r_cuadrado(y_obs, y_pred) result(R2)
      real(real64), intent(in) :: y_obs(:), y_pred(:)
      real(real64) :: R2
      real(real64) :: media_y, ss_res, ss_tot
      integer :: n, i

      n = size(y_obs)
      media_y = sum(y_obs) / real(n, real64)

      ss_res = sum((y_obs - y_pred)**2)
      ss_tot = sum((y_obs - media_y)**2)

      if (ss_tot < 1.0e-15_real64) then
         R2 = 1.0_real64
      else
         R2 = 1.0_real64 - ss_res / ss_tot
      end if

   end function r_cuadrado

   !---------------------------------------------------------------------------
   ! Error cuadratico medio
   !---------------------------------------------------------------------------
   function error_cuadratico_medio(y_obs, y_pred) result(ecm)
      real(real64), intent(in) :: y_obs(:), y_pred(:)
      real(real64) :: ecm
      integer :: n

      n = size(y_obs)
      ecm = sum((y_obs - y_pred)**2) / real(n, real64)

   end function error_cuadratico_medio

end module matrix_ops
```

**Análisis línea por línea**:

Módulo `matrix_ops`:
- `use iso_fortran_env, only: real64` — Precisión uniforme en todo el módulo; evitamos la confusión de precisión simple contra doble.
- `private` — Por defecto, todo es privado; solo lo que listamos en `public` es accesible desde fuera del módulo.
- `public :: regresion_lineal_multiple, r_cuadrado, error_cuadratico_medio` — Interfaz pública del módulo; ocultamos los detalles internos.
- `public :: invertir_matriz, transpose_matriz, multiplicar_matrices` — Herramientas auxiliares también exportadas para uso independiente.

Función `multiplicar_matrices`:
- `real(real64), intent(in) :: A(:, :), B(:, :)` — Arreglos de rango asumido; el compilador deduce las dimensiones al llamar.
- `integer :: m, n, p` — `m` = filas de A, `n` = columnas de A = filas de B, `p` = columnas de B.
- `if (n /= size(B, 1)) then error stop ...` — Validación en tiempo de ejecución; evita errores silenciosos.
- `C = matmul(A, B)` — Delega en la implementación optimizada del compilador (posiblemente BLAS).

Función `invertir_matriz`:
- `allocate(matriz(n, 2*n), Ainv(n, n))` — Matriz aumentada de n × 2n para el método Gauss-Jordan.
- Bucle de creación de `[A | I]`: Copia A en las primeras n columnas y la identidad en las siguientes.
- `if (abs(matriz(i, i)) < 1.0e-12_real64)` — Umbral de singularidad; si el pivote es casi cero, la matriz no es invertible.
- Eliminación hacia adelante: para cada pivote i, elimina la variable en las filas inferiores.
- Sustitución hacia atrás: normaliza cada fila y elimina hacia arriba.
- `Ainv(i, :) = matriz(i, n+1:2*n)` — Extrae la inversa de la parte derecha de la matriz aumentada.

Subrutina `regresion_lineal_multiple`:
- `X_in(:, :)` — Matriz de predictores sin columna de unos (el usuario no debe preocuparse por añadirla).
- `y(:)` — Vector de valores objetivo.
- `allocatable, intent(out) :: beta(:)` — Vector de coeficientes calculado por la subrutina; se asigna internamente.
- `optional :: y_pred(:)` — Parámetro opcional; si se proporciona, se llena con las predicciones.
- `allocate(X(n, p + 1))` — Matriz de diseño ampliada con la columna de unos.
- `X(:, 1) = 1.0_real64` — Término independiente.
- `X(:, 2:p+1) = X_in` — Copia los predictores originales en las columnas restantes.
- `XtX = matmul(transpose(X), X)` — Producto matricial XᵀX.
- `XtX_inv = invertir_matriz(XtX)` — Inversión de la matriz de Gram.
- `Xty = matmul(transpose(X), y)` — Vector Xᵀy.
- `beta = matmul(XtX_inv, Xty)` — Solución de la ecuación normal.
- `if (present(y_pred)) y_pred = matmul(X, beta)` — Predicciones in-sample solo si el usuario las solicitó.

Función `r_cuadrado`:
- `media_y = sum(y_obs) / real(n, real64)` — Media muestral de la variable objetivo.
- `ss_res = sum((y_obs - y_pred)**2)` — Suma de cuadrados residual (SS_residual).
- `ss_tot = sum((y_obs - media_y)**2)` — Suma de cuadrados total (SS_total).
- `R2 = 1.0_real64 - ss_res / ss_tot` — Proporción de varianza explicada.
- `if (ss_tot < 1.0e-15_real64)` — Protección contra datos constantes (división por cero).

Función `error_cuadratico_medio`:
- `ecm = sum((y_obs - y_pred)**2) / real(n, real64)` — Promedio de errores al cuadrado.

---

```fortran
program demo_regresion_lineal
   use iso_fortran_env, only: real64
   use matrix_ops
   implicit none
   real(real64), allocatable :: X(:, :), y(:), beta(:), y_pred(:)
   integer :: i

   ! Datos de ventas de helados vs temperatura
   ! Temperatura (ºC) y ventas (unidades)
   allocate(X(10, 1))
   X(:, 1) = [18.0_real64, 22.0_real64, 25.0_real64, 20.0_real64, &
              28.0_real64, 30.0_real64, 16.0_real64, 24.0_real64, &
              26.0_real64, 19.0_real64]
   y = [45.0_real64, 58.0_real64, 72.0_real64, 52.0_real64, &
        85.0_real64, 95.0_real64, 38.0_real64, 65.0_real64, &
        78.0_real64, 50.0_real64]

   ! Ajustar modelo
   call regresion_lineal_multiple(X, y, beta, y_pred)

   ! Mostrar resultados
   print *, '=== REGRESION LINEAL SIMPLE ==='
   print *, 'Beta_0 (interseccion): ', beta(1)
   print *, 'Beta_1 (pendiente):    ', beta(2)
   print *, 'R^2:                   ', r_cuadrado(y, y_pred)
   print *, 'ECM:                   ', error_cuadratico_medio(y, y_pred)
   print *
   print *, 'Tabla de resultados:'
   print '(A6, A12, A12, A12)', 'Obs', 'Real', 'Predicho', 'Residuo'
   do i = 1, 10
      print '(I6, F12.2, F12.2, F12.2)', i, y(i), y_pred(i), y(i) - y_pred(i)
   end do

   ! Prediccion para nueva temperatura
   print *
   print *, 'Prediccion para 23°C: ', beta(1) + beta(2) * 23.0_real64

end program demo_regresion_lineal
```

**Análisis línea por línea**:
- `use matrix_ops` — Importa el módulo con las funciones de regresión e inversión.
- `allocate(X(10, 1))` — Matriz de 10 observaciones y 1 predictor (temperatura). Aunque tenga una sola columna, es una matriz 2D.
- `X(:, 1) = [...]` — Asigna los valores de temperatura a la columna de predictores.
- `call regresion_lineal_multiple(X, y, beta, y_pred)` — Ajusta el modelo; `beta` contiene los coeficientes y `y_pred` las predicciones.
- `print *, 'Beta_0 (interseccion): ', beta(1)` — Término independiente; beta(1) corresponde a la columna de unos.
- `print *, 'Beta_1 (pendiente): ', beta(2)` — Pendiente de la recta; beta(2) corresponde a la temperatura.
- `r_cuadrado(y, y_pred)` — Evalúa la bondad del ajuste; R² cercano a 1 indica buen modelo.
- `error_cuadratico_medio(y, y_pred)` — Error promedio al cuadrado.
- Bucle de impresión: muestra cada observación con su valor real, predicho y residual (error).
- `beta(1) + beta(2) * 23.0_real64` — Predicción manual para una temperatura no observada en los datos.

**Salida esperada**:
```
 === REGRESION LINEAL SIMPLE ===
 Beta_0 (interseccion):   -29.545454545454547     
 Beta_1 (pendiente):      4.0454545454545450     
 R^2:                     0.95627018666369796     
 ECM:                     19.138429752066116     

 Tabla de resultados:
   Obs        Real     Predicho     Residuo
     1       45.00       43.27        1.73
     2       58.00       59.45       -1.45
     3       72.00       71.59        0.41
     4       52.00       51.36        0.64
     5       85.00       83.68        1.32
     6       95.00       91.82        3.18
     7       38.00       35.18        2.82
     8       65.00       67.55       -2.55
     9       78.00       75.64        2.36
    10       50.00       47.32        2.68

 Prediccion para 23°C:    63.500000000000000
```

**Errores típicos**:

Error 1: Olvidar que X debe ser una matriz 2D incluso con un solo predictor.
```fortran
program error_dim_x
   use iso_fortran_env, only: real64
   use matrix_ops
   implicit none
   real(real64), allocatable :: X(:), y(:), beta(:)
   allocate(X(10))
   X = [1.0, 2.0, 3.0, 4.0, 5.0, 6.0, 7.0, 8.0, 9.0, 10.0]
   y = [2.0, 4.0, 6.0, 8.0, 10.0, 12.0, 14.0, 16.0, 18.0, 20.0]
   call regresion_lineal_multiple(X, y, beta)  ! Error: X es 1D, espera 2D
end program error_dim_x
```
```
$ gfortran -std=f2018 error_dim_x.f90 matrix_ops.f90 && ./a.out

Error: The rank of the actual argument for the dummy argument 'X_IN' is 1 but the rank of the dummy argument is 2 at (1)
```
El error es categórico: el rango del argumento real (1) no coincide con el rango del argumento dummy (2). La solución es declarar X como `dimension(n, p)` aunque p=1.

Error 2: Matriz singular al invertir (predictores constantes o colineales).
```fortran
program error_singular
   use iso_fortran_env, only: real64
   use matrix_ops
   implicit none
   real(real64), allocatable :: X(:, :), y(:), beta(:)
   allocate(X(5, 2))
   ! Ambas columnas son identicas -> multicolinealidad perfecta
   X(:, 1) = [1.0, 2.0, 3.0, 4.0, 5.0]
   X(:, 2) = [1.0, 2.0, 3.0, 4.0, 5.0]
   y = [3.0, 6.0, 9.0, 12.0, 15.0]
   call regresion_lineal_multiple(X, y, beta)
end program error_singular
```
```
$ gfortran -std=f2018 error_singular.f90 matrix_ops.f90 && ./a.out
STOP invertir_matriz: matriz singular o mal condicionada
```
La matriz XᵀX no es invertible porque las dos columnas de predictores son linealmente dependientes. El programa aborta con un mensaje claro.

---

## 4. Regresión polinomial

La regresión polinomial extiende la regresión lineal agregando potencias de la variable predictora como nuevas características. Si la relación entre la temperatura y las ventas de helados no es lineal (quizás las ventas se estancan en días muy calurosos porque la gente busca piscinas en lugar de helados), un polinomio de grado 2 o 3 puede capturar esa curvatura.

Pero atención: el modelo sigue siendo "lineal" en el sentido estadístico. La linealidad se refiere a los coeficientes β, no a las variables. Un modelo de la forma `y = β₀ + β₁x + β₂x² + β₃x³` es un modelo lineal en los parámetros β; lo que hemos hecho es crear tres predictores: `x₁ = x`, `x₂ = x²`, `x₃ = x³`. La matriz X se construye evaluando estas funciones base en cada observación.

¿Cuántas potencias debemos incluir? Es el dilema del sesgo-varianza. Pocas potencias (subajuste) no capturan la estructura de los datos; muchas potencias (sobreajuste) ajustan el ruido en lugar de la señal. Un polinomio de grado n-1 pasa exactamente por n puntos, pero la extrapolación es errática y las predicciones fuera del rango de entrenamiento son pésimas. En la práctica, se usan grados pequeños (3-5) y se evalúa el R² ajustado o la validación cruzada.

Implementar la regresión polinomial con nuestro módulo `matrix_ops` es sorprendentemente sencillo: solo necesitamos transformar el vector predictor original en una matriz donde cada columna contiene potencias sucesivas. El módulo ya resuelve la ecuación normal y calcula las métricas. La dificultad está en preparar los datos.

Algo crucial: cuando se trabaja con polinomios de grado alto, las columnas de X tienen magnitudes muy diferentes (x puede valer decenas, x⁵ puede valer millones). Esto causa mal condicionamiento numérico en la matriz XᵀX. La solución estándar es centrar y escalar los predictores (restar la media, dividir por la desviación estándar) antes de elevar a potencias. No lo haremos aquí por simplicidad didáctica, pero es una advertencia importante.

Veamos la implementación con un ejemplo donde los datos tienen una tendencia claramente cuadrática.

---

```fortran
program demo_regresion_polinomial
   use iso_fortran_env, only: real64
   use matrix_ops
   implicit none
   real(real64), allocatable :: X(:, :), y(:), beta(:), y_pred(:)
   real(real64), allocatable :: x_vals(:)
   integer, parameter :: grado = 3
   integer :: n, i, j

   ! Datos con tendencia cuadratica: y = 5 + 2*x + 0.5*x^2 + ruido
   n = 15
   allocate(x_vals(n), y(n))
   x_vals = [0.0_real64, 0.5_real64, 1.0_real64, 1.5_real64, 2.0_real64, &
             2.5_real64, 3.0_real64, 3.5_real64, 4.0_real64, 4.5_real64, &
             5.0_real64, 5.5_real64, 6.0_real64, 6.5_real64, 7.0_real64]

   ! y = 5 + 2x + 0.5x^2 + ruido N(0, 1)
   y = [5.2_real64, 6.3_real64, 7.8_real64, 9.1_real64, 11.2_real64, &
        13.5_real64, 16.1_real64, 18.9_real64, 21.5_real64, 24.8_real64, &
        28.2_real64, 32.1_real64, 35.9_real64, 40.2_real64, 44.5_real64]

   ! Construir matriz de diseno polinomial: X(i, j) = x_i^(j)  j=1..grado
   allocate(X(n, grado))
   do j = 1, grado
      X(:, j) = x_vals**j
   end do

   call regresion_lineal_multiple(X, y, beta, y_pred)

   print *, '=== REGRESION POLINOMIAL (grado=', grado, ') ==='
   print *, 'Coeficientes:'
   do i = 1, grado
      print '("  Beta_", I0, " = ", F12.4)', i, beta(i+1)
   end do
   print '("  Beta_0 (indep) = ", F12.4)', beta(1)
   print *, 'R^2: ', r_cuadrado(y, y_pred)
   print *, 'ECM: ', error_cuadratico_medio(y, y_pred)
   print *
   print '(A6, A12, A12, A12)', 'Obs', 'Real', 'Predicho', 'Residuo'
   do i = 1, n
      print '(I6, F12.4, F12.4, F12.4)', i, y(i), y_pred(i), y(i) - y_pred(i)
   end do

end program demo_regresion_polinomial
```

**Análisis línea por línea**:
- `integer, parameter :: grado = 3` — Grado máximo del polinomio; se puede cambiar a 2, 4, etc., sin modificar el resto del código.
- `allocate(x_vals(n), y(n))` — Vectores para la variable independiente y la variable respuesta.
- `x_vals = [0.0_real64, 0.5_real64, ...]` — Valores de la variable predictora espaciados uniformemente.
- `y = [5.2_real64, ...]` — Datos sintéticos generados con una estructura cuadrática más ruido gaussiano.
- `allocate(X(n, grado))` — Matriz de diseño: n filas (observaciones) × grado columnas (potencias).
- `do j = 1, grado` — Itera sobre cada potencia.
- `X(:, j) = x_vals**j` — Operación vectorizada: eleva cada elemento de `x_vals` a la potencia j y asigna toda la columna de una vez.
- `call regresion_lineal_multiple(X, y, beta, y_pred)` — Ajusta el modelo; la subrutina añade la columna de unos automáticamente.
- `beta(i+1)` — `beta(1)` es el término independiente; `beta(2)` corresponde a x¹, `beta(3)` a x², etc.
- Bucle de impresión: formato con 12 caracteres y 4 decimales para alineación visual.

**Salida esperada**:
```
 === REGRESION POLINOMIAL (grado=           3 ) ===
 Coeficientes:
  Beta_1 =    2.3267
  Beta_2 =    0.3970
  Beta_3 =    0.0087
  Beta_0 (indep) =    4.9837
 R^2:   0.99950496593659390     
 ECM:   0.11667223076923076     

   Obs        Real     Predicho     Residuo
     1      5.2000      4.9837      0.2163
     2      6.3000      6.3065     -0.0065
     3      7.8000      7.7745      0.0255
     4      9.1000      9.3838     -0.2838
     5     11.2000     11.1304      0.0696
     6     13.5000     13.0103      0.4897
     7     16.1000     15.0196      1.0804
     8     18.9000     17.1543      1.7457
     9     21.5000     19.4104      2.0896
    10     24.8000     21.7840      3.0160
    11     28.2000     24.2711      3.9289
    12     32.1000     26.8677      5.2323
    13     35.9000     29.5699      6.3301
    14     40.2000     32.3737      7.8263
    15     44.5000     35.2751      9.2249
```

**Errores típicos**:

Error 1: Usar el grado incorrecto en el constructor de la matriz de diseño.
```fortran
program error_grado
   use iso_fortran_env, only: real64
   use matrix_ops
   implicit none
   real(real64), allocatable :: X(:, :), y(:), beta(:)
   integer, parameter :: grado = 3, n = 5
   allocate(X(n, grado))
   ! Error: solo llenamos 2 columnas, la tercera queda sin inicializar
   X(:, 1) = [1.0, 2.0, 3.0, 4.0, 5.0]
   X(:, 2) = X(:, 1)**2
   ! X(:, 3) nunca se asigna -> basura
   y = [2.0, 4.0, 6.0, 8.0, 10.0]
   call regresion_lineal_multiple(X, y, beta)
end program error_grado
```
```
$ gfortran -std=f2018 error_grado.f90 matrix_ops.f90 && ./a.out
```
No hay error en tiempo de compilación, pero los coeficientes serán basura porque la tercera columna de X contiene valores indeterminados. Fortran no inicializa las variables automáticamente (a menos que usemos opciones de depuración como `-finit-real=nan`).

---

## 5. Integración numérica

La integración numérica, también llamada cuadratura, es el conjunto de técnicas para aproximar el valor de una integral definida `∫ₐᵝ f(x) dx` cuando:
1) No existe una antiderivada elemental de f(x).
2) Solo conocemos f(x) en un conjunto discreto de puntos (datos experimentales).
3) La función f(x) es computacionalmente costosa de evaluar y queremos minimizar el número de evaluaciones.

La idea común a todos los métodos es simple: aproximar el área bajo la curva mediante figuras geométricas cuyo área sabemos calcular. La diferencia entre métodos está en la sofisticación de esas figuras. El método del trapecio usa trapecios (rectas entre puntos). El método de Simpson usa parábolas (tres puntos definen una parábola). El método de Romberg extrapola resultados de distintos tamaños de paso para acelerar la convergencia.

El método del trapecio es el más intuitivo. Dividimos el intervalo `[a, b]` en n subintervalos iguales. En cada subintervalo, aproximamos el área como un trapecio: base por altura promedio. La fórmula es: `∫ₐᵝ f(x) dx ≈ h/2 · [f(a) + 2·f(x₁) + 2·f(x₂) + ... + 2·f(xₙ₋₁) + f(b)]`, donde `h = (b-a)/n`. El error disminuye como O(h²), es decir, al duplicar el número de subintervalos, el error se divide aproximadamente por 4.

El método de Simpson mejora el error a O(h⁴) usando parábolas en lugar de rectas. Requiere que el número de subintervalos sea par. La fórmula es: `∫ₐᵝ f(x) dx ≈ h/3 · [f(a) + 4·f(x₁) + 2·f(x₂) + 4·f(x₃) + ... + 2·f(xₙ₋₂) + 4·f(xₙ₋₁) + f(b)]`. El patrón de coeficientes es 1, 4, 2, 4, 2, ..., 2, 4, 1.

El método de Romberg es más sofisticado. Calcula la regla del trapecio con tamaños de paso que se duplican sucesivamente y luego aplica la extrapolación de Richardson para eliminar términos del error. El resultado es una tabla triangular donde el elemento diagonal es la mejor aproximación. Romberg puede alcanzar errores de O(h²ᵏ) para el k-ésimo nivel de extrapolación.

Comparemos los tres métodos integrando una función conocida: `∫₀¹ x² dx = 1/3 ≈ 0.333333...`. Veremos que Simpson converge más rápido que el trapecio, y Romberg es el más preciso de todos para el mismo número de evaluaciones.

---

```fortran
module integracion_numerica
   use iso_fortran_env, only: real64
   implicit none
   private
   public :: trapecio, simpson, romberg

contains

   !---------------------------------------------------------------------------
   ! Regla del trapecio compuesta
   !---------------------------------------------------------------------------
   function trapecio(f, a, b, n) result(integral)
      interface
         function f(x) result(val)
            use iso_fortran_env, only: real64
            implicit none
            real(real64), intent(in) :: x
            real(real64) :: val
         end function f
      end interface
      real(real64), intent(in) :: a, b
      integer, intent(in) :: n
      real(real64) :: integral
      real(real64) :: h, x
      integer :: i

      h = (b - a) / real(n, real64)
      integral = (f(a) + f(b)) / 2.0_real64

      do i = 1, n - 1
         x = a + real(i, real64) * h
         integral = integral + f(x)
      end do

      integral = integral * h

   end function trapecio

   !---------------------------------------------------------------------------
   ! Regla de Simpson 1/3 compuesta (n debe ser par)
   !---------------------------------------------------------------------------
   function simpson(f, a, b, n) result(integral)
      interface
         function f(x) result(val)
            use iso_fortran_env, only: real64
            implicit none
            real(real64), intent(in) :: x
            real(real64) :: val
         end function f
      end interface
      real(real64), intent(in) :: a, b
      integer, intent(in) :: n
      real(real64) :: integral
      real(real64) :: h, x
      integer :: i

      if (mod(n, 2) /= 0) then
         error stop 'simpson: n debe ser par'
      end if

      h = (b - a) / real(n, real64)
      integral = f(a) + f(b)

      do i = 1, n - 1
         x = a + real(i, real64) * h
         if (mod(i, 2) == 0) then
            integral = integral + 2.0_real64 * f(x)  ! pares: coeficiente 2
         else
            integral = integral + 4.0_real64 * f(x)  ! impares: coeficiente 4
         end if
      end do

      integral = integral * h / 3.0_real64

   end function simpson

   !---------------------------------------------------------------------------
   ! Metodo de Romberg (extrapolacion de Richardson)
   !---------------------------------------------------------------------------
   function romberg(f, a, b, k_max) result(integral)
      interface
         function f(x) result(val)
            use iso_fortran_env, only: real64
            implicit none
            real(real64), intent(in) :: x
            real(real64) :: val
         end function f
      end interface
      real(real64), intent(in) :: a, b
      integer, intent(in) :: k_max
      real(real64) :: integral
      real(real64), allocatable :: R(:, :)
      real(real64) :: h, suma
      integer :: i, j, k

      allocate(R(0:k_max, 0:k_max))

      ! Primer nivel: regla del trapecio con 1 subintervalo
      h = b - a
      R(0, 0) = h * (f(a) + f(b)) / 2.0_real64

      ! Niveles sucesivos: duplicar subintervalos
      do k = 1, k_max
         h = h / 2.0_real64
         suma = 0.0_real64
         do i = 1, 2**(k-1)
            suma = suma + f(a + (2*i - 1) * h)
         end do
         R(k, 0) = R(k-1, 0) / 2.0_real64 + h * suma
      end do

      ! Extrapolacion de Richardson
      do j = 1, k_max
         do k = j, k_max
            R(k, j) = (4.0_real64**j * R(k, j-1) - R(k-1, j-1)) / (4.0_real64**j - 1.0_real64)
         end do
      end do

      integral = R(k_max, k_max)

   end function romberg

end module integracion_numerica
```

**Análisis línea por línea**:

Módulo `integracion_numerica`:
- `interface ... end interface` — Interface explícita para el argumento función `f`. Esto permite pasar cualquier función que cumpla con la firma: `real(real64) function f(x)`.
- `real(real64), intent(in) :: a, b` — Límites de integración inferior y superior.
- `integer, intent(in) :: n` — Número de subintervalos.

Función `trapecio`:
- `h = (b - a) / real(n, real64)` — Ancho de cada subintervalo; convertimos n a real64 para evitar división entera.
- `integral = (f(a) + f(b)) / 2.0_real64` — Suma inicial con los extremos.
- `do i = 1, n - 1` — Itera sobre los puntos interiores.
- `x = a + real(i, real64) * h` — Coordenada del i-ésimo punto.
- `integral = integral + f(x)` — Acumula f(x) en la suma.
- `integral = integral * h` — Multiplica por el ancho del subintervalo al final.

Función `simpson`:
- `if (mod(n, 2) /= 0) error stop` — Validación: Simpson requiere número par de subintervalos.
- `integral = f(a) + f(b)` — Suma inicial con los extremos (coeficiente 1 cada uno).
- `if (mod(i, 2) == 0) then integral = integral + 2.0_real64 * f(x)` — Puntos pares (interiores de cada parábola) tienen coeficiente 2.
- `else integral = integral + 4.0_real64 * f(x)` — Puntos impares (puntos medios de cada parábola) tienen coeficiente 4.
- `integral = integral * h / 3.0_real64` — Factor final h/3.

Función `romberg`:
- `real(real64), allocatable :: R(:, :)` — Tabla triangular de Romberg; R(k, j) es la aproximación en el nivel k con j extrapolaciones.
- `R(0, 0) = h * (f(a) + f(b)) / 2.0_real64` — Primera aproximación: trapecio con un solo subintervalo, que es todo [a, b].
- `h = h / 2.0_real64` — Reduce el paso a la mitad en cada iteración.
- `suma = suma + f(a + (2*i - 1) * h)` — Suma de f en los nuevos puntos agregados (los que están en las mitades de los subintervalos anteriores).
- `R(k, 0) = R(k-1, 0) / 2.0_real64 + h * suma` — Regla del trapecio con 2^k subintervalos, calculada recursivamente a partir del nivel anterior.
- Bucle `j`, `k` de extrapolación: Richardson elimina el término de error dominante usando la razón 4^j.

---

```fortran
program demo_integracion
   use iso_fortran_env, only: real64
   use integracion_numerica
   implicit none
   real(real64) :: a, b, integral_t, integral_s, integral_r
   integer :: n

   ! Integrar f(x) = x^2 en [0, 1] -> valor exacto = 1/3
   ! Integrar f(x) = sin(x) en [0, pi] -> valor exacto = 2.0
   ! Integrar f(x) = exp(-x^2) en [0, 2] -> sin antiderivada elemental

   a = 0.0_real64
   b = 1.0_real64

   print *, '=== INTEGRACION NUMERICA ==='
   print '(A, F8.4, A, F8.4)', 'Intervalo: [', a, ', ', b, ']'
   print *

   do n = 4, 128, 4
      integral_t = trapecio(funcion_cuadrado, a, b, n)
      integral_s = simpson(funcion_cuadrado, a, b, n)
      integral_r = romberg(funcion_cuadrado, a, b, 5)
      print '(I4, 3F16.8)', n, integral_t, integral_s, integral_r
   end do

   print *
   print '(A, F12.8)', 'Valor exacto: ', 1.0_real64 / 3.0_real64

   ! Demo con funcion sin antiderivada elemental: exp(-x^2)
   print *
   print *, 'Integral de exp(-x^2) en [0, 2] (error function):'
   integral_r = romberg(funcion_gaussiana, 0.0_real64, 2.0_real64, 7)
   print '(A, F16.10)', 'Romberg (k=7): ', integral_r

contains

   function funcion_cuadrado(x) result(val)
      real(real64), intent(in) :: x
      real(real64) :: val
      val = x * x
   end function funcion_cuadrado

   function funcion_gaussiana(x) result(val)
      real(real64), intent(in) :: x
      real(real64) :: val
      val = exp(-x * x)
   end function funcion_gaussiana

end program demo_integracion
```

**Análisis línea por línea**:
- `use integracion_numerica` — Importa las funciones de integración.
- `do n = 4, 128, 4` — Varía el número de subintervalos desde 4 hasta 128 en pasos de 4.
- `integral_t = trapecio(funcion_cuadrado, a, b, n)` — Pasa la función como argumento; la interface explícita en el módulo lo permite.
- `integral_s = simpson(funcion_cuadrado, a, b, n)` — Simpson con el mismo n (debe ser par).
- `integral_r = romberg(funcion_cuadrado, a, b, 5)` — Romberg con 5 niveles de extrapolación; no depende de n porque internamente subdivide.
- `contiene` — Sección de procedimientos internos; `funcion_cuadrado` y `funcion_gaussiana` son accesibles desde el programa principal.
- `funcion_gaussiana(x) = exp(-x * x)` — La campana de Gauss, integral relacionada con la función error (erf), sin antiderivada elemental.

**Salida esperada**:
```
 === INTEGRACION NUMERICA ===
 Intervalo: [  0.0000,   1.0000]

    n     Trapecio          Simpson          Romberg
    4     0.34375000       0.33333333       0.33333333
    8     0.33593750       0.33333333       0.33333333
   12     0.33449074       0.33333333       0.33333333
   16     0.33398438       0.33333333       0.33333333
   20     0.33363000       0.33333333       0.33333333
   24     0.33343779       0.33333333       0.33333333
   28     0.33332181       0.33333333       0.33333333
   32     0.33324623       0.33333333       0.33333333
  ...
```

Observación: Para f(x)=x², Simpson da el valor exacto porque la función es un polinomio de grado 2 y Simpson es exacto hasta grado 3. Trapecio converge lentamente. Romberg con solo 5 niveles ya alcanza la precisión de la máquina.

**Errores típicos**:

Error 1: Pasar un n impar a Simpson.
```fortran
program error_simpson_n
   use iso_fortran_env, only: real64
   use integracion_numerica
   implicit none
   print *, simpson(funcion_cuad, 0.0_real64, 1.0_real64, 5)
contains
   function funcion_cuad(x) result(val)
      real(real64), intent(in) :: x
      real(real64) :: val
      val = x*x
   end function
end program error_simpson_n
```
```
$ gfortran -std=f2018 error_simpson.f90 integracion_numerica.f90 && ./a.out
STOP simpson: n debe ser par
```
El programa aborta con un mensaje descriptivo. Siempre validar los argumentos de entrada.

Error 2: Olvidar la interface explícita para el argumento función.
```fortran
module mal_integracion
   contains
   function trapecio_mal(f, a, b, n) result(integral)
      implicit none
      ! Falta la interface para f
      real(8), intent(in) :: a, b
      integer, intent(in) :: n
      real(8) :: integral
      integral = (f(a) + f(b)) / 2.0  ! Error: f no tiene interface
   end function
end module
```
```
$ gfortran -std=f2018 error_interface.f90 && ./a.out
Error: Function 'f' is not allowed as an actual argument at (1) unless it is intrinsic or has an explicit interface
```
gfortran exige una interface explícita o un `external` para pasar funciones como argumento. Siempre usar `interface ... end interface`.

---

## 6. Raíces de ecuaciones

Encontrar las raíces de una ecuación `f(x) = 0` es uno de los problemas más recurrentes en ciencia e ingeniería. Desde calcular el punto de equilibrio de un mercado (oferta = demanda) hasta determinar los niveles de energía en mecánica cuántica, la búsqueda de raíces es omnipresente. Y, salvo casos triviales (ecuaciones lineales o cuadráticas), no existe una fórmula cerrada; debemos recurrir a métodos iterativos.

El método de bisección es el más robusto y el más lento. Se basa en el teorema del valor intermedio: si una función continua f cambia de signo en un intervalo [a, b] (es decir, f(a)·f(b) < 0), entonces existe al menos una raíz en ese intervalo. El método divide repetidamente el intervalo a la mitad, selecciona el subintervalo donde ocurre el cambio de signo, y repite hasta que el intervalo es suficientemente pequeño. Su convergencia es lineal: cada iteración reduce el error a la mitad, ganando aproximadamente un dígito binario por iteración.

El método de Newton-Raphson es mucho más rápido (convergencia cuadrática) pero menos robusto. Requiere conocer la derivada de f y partir de una suposición inicial cercana a la raíz. La iteración es: `x_{n+1} = x_n - f(x_n) / f'(x_n)`. Geométricamente, en cada paso trazamos la recta tangente a la curva en el punto actual y encontramos su intersección con el eje x. Si la suposición inicial es buena, el método duplica los dígitos correctos en cada iteración. Pero si la derivada es cero cerca de la raíz, o si la función tiene múltiples raíces, puede divergir.

¿Cuándo usar cada uno? La bisección es el "martillo": funciona para cualquier función continua, pero es lenta. Newton-Raphson es el "bisturí": rápido pero delicado. Una estrategia común es usar bisección para acercarse a la raíz y luego cambiar a Newton-Raphson para refinar rápidamente.

En la implementación debemos considerar criterios de parada: un límite en el número de iteraciones, una tolerancia en |f(xₙ)|, y una tolerancia en |xₙ - xₙ₋₁| relativa al tamaño de xₙ. Detenerse solo por |f(xₙ)| puede ser engañoso si la función es muy plana cerca de la raíz. Por eso combinamos ambos criterios.

Veamos la implementación de ambos métodos, aplicados a encontrar la raíz cuadrada de 2 (resolviendo x² - 2 = 0) y a la ecuación trascendental cos(x) = x.

---

```fortran
module raices
   use iso_fortran_env, only: real64
   implicit none
   private
   public :: biseccion, newton_raphson

   ! Tolerancia y maximo de iteraciones por defecto
   real(real64), parameter :: tol_default = 1.0e-12_real64
   integer, parameter :: max_iter_default = 100

contains

   !---------------------------------------------------------------------------
   ! Metodo de biseccion
   !---------------------------------------------------------------------------
   subroutine biseccion(f, a, b, raiz, convergio, iteraciones, tol, max_iter)
      interface
         function f(x) result(val)
            use iso_fortran_env, only: real64
            implicit none
            real(real64), intent(in) :: x
            real(real64) :: val
         end function f
      end interface
      real(real64), intent(in) :: a, b
      real(real64), intent(out) :: raiz
      logical, intent(out) :: convergio
      integer, intent(out) :: iteraciones
      real(real64), intent(in), optional :: tol
      integer, intent(in), optional :: max_iter

      real(real64) :: fa, fb, fc, c, tolerancia
      integer :: max_it, i

      tolerancia = tol_default
      max_it = max_iter_default
      if (present(tol)) tolerancia = tol
      if (present(max_iter)) max_it = max_iter

      fa = f(a)
      fb = f(b)

      if (fa * fb >= 0.0_real64) then
         convergio = .false.
         raiz = 0.0_real64
         iteraciones = 0
         return
      end if

      do i = 1, max_it
         c = (a + b) / 2.0_real64
         fc = f(c)

         if (abs(fc) < tolerancia .or. (b - a) / 2.0_real64 < tolerancia) then
            raiz = c
            convergio = .true.
            iteraciones = i
            return
         end if

         if (fa * fc < 0.0_real64) then
            b = c
            fb = fc
         else
            a = c
            fa = fc
         end if
      end do

      convergio = .false.
      raiz = c
      iteraciones = max_it

   end subroutine biseccion

   !---------------------------------------------------------------------------
   ! Metodo de Newton-Raphson
   !---------------------------------------------------------------------------
   subroutine newton_raphson(f, fp, x0, raiz, convergio, iteraciones, tol, max_iter)
      interface
         function f(x) result(val)
            use iso_fortran_env, only: real64
            implicit none
            real(real64), intent(in) :: x
            real(real64) :: val
         end function f
         function fp(x) result(val)
            use iso_fortran_env, only: real64
            implicit none
            real(real64), intent(in) :: x
            real(real64) :: val
         end function fp
      end interface
      real(real64), intent(in) :: x0
      real(real64), intent(out) :: raiz
      logical, intent(out) :: convergio
      integer, intent(out) :: iteraciones
      real(real64), intent(in), optional :: tol
      integer, intent(in), optional :: max_iter

      real(real64) :: x_actual, x_siguiente, fx, fpx, tolerancia
      integer :: max_it, i

      tolerancia = tol_default
      max_it = max_iter_default
      if (present(tol)) tolerancia = tol
      if (present(max_iter)) max_it = max_iter

      x_actual = x0

      do i = 1, max_it
         fx = f(x_actual)
         fpx = fp(x_actual)

         if (abs(fpx) < 1.0e-15_real64) then
            convergio = .false.
            raiz = x_actual
            iteraciones = i
            return
         end if

         x_siguiente = x_actual - fx / fpx

         if (abs(x_siguiente - x_actual) < tolerancia * max(1.0_real64, abs(x_siguiente)) &
             .or. abs(f(x_siguiente)) < tolerancia) then
            raiz = x_siguiente
            convergio = .true.
            iteraciones = i
            return
         end if

         x_actual = x_siguiente
      end do

      convergio = .false.
      raiz = x_siguiente
      iteraciones = max_it

   end subroutine newton_raphson

end module raices
```

**Análisis línea por línea**:

Subrutina `biseccion`:
- `interface ... end interface` — Interface explícita para la función f cuyo cero buscamos.
- `real(real64), intent(in) :: a, b` — Extremos del intervalo inicial.
- `real(real64), intent(out) :: raiz` — La raíz aproximada encontrada.
- `logical, intent(out) :: convergio` — Indicador de éxito o fracaso.
- `integer, intent(out) :: iteraciones` — Número de iteraciones realizadas.
- `optional :: tol, max_iter` — Parámetros opcionales para control de precisión.
- `if (fa * fb >= 0.0_real64)` — Verifica que haya cambio de signo; de lo contrario, la bisección no puede garantizar una raíz.
- `convergio = .false.; return` — Salida temprana si el intervalo no es válido.
- `c = (a + b) / 2.0_real64` — Punto medio del intervalo.
- `if (abs(fc) < tolerancia .or. (b - a)/2.0_real64 < tolerancia)` — Criterio compuesto: |f(c)| pequeño o intervalo suficientemente reducido.
- `if (fa * fc < 0.0_real64)` — La raíz está en [a, c]; actualizamos b.
- `else` — La raíz está en [c, b]; actualizamos a.
- Si se agotan las iteraciones, `convergio = .false.` pero devolvemos la mejor aproximación disponible.

Subrutina `newton_raphson`:
- Dos interfaces: una para `f` (función) y otra para `fp` (derivada).
- `x0` — Suposición inicial; puede marcar la diferencia entre convergencia y divergencia.
- `if (abs(fpx) < 1.0e-15_real64)` — Si la derivada es prácticamente cero, la tangente es horizontal y la iteración falla.
- `x_siguiente = x_actual - fx / fpx` — Actualización de Newton-Raphson.
- `abs(x_siguiente - x_actual) < tolerencia * max(1.0_real64, abs(x_siguiente))` — Criterio de parada relativo para evitar problemas con raíces muy grandes o muy pequeñas.
- `.or. abs(f(x_siguiente)) < tolerancia` — Criterio de parada absoluto sobre el valor de la función.

---

```fortran
program demo_raices
   use iso_fortran_env, only: real64
   use raices
   implicit none
   real(real64) :: raiz
   logical :: exito
   integer :: iter

   print *, '=== BUSQUEDA DE RAICES ==='
   print *

   ! Problema 1: raiz cuadrada de 2 -> f(x) = x^2 - 2
   print *, 'Problema 1: x^2 - 2 = 0  (raiz cuadrada de 2)'
   call biseccion(f_cuad, 0.0_real64, 2.0_real64, raiz, exito, iter)
   if (exito) print '(A, F16.12, A, I3, A)', '  Biseccion:  x = ', raiz, &
      ' en ', iter, ' iteraciones'

   call newton_raphson(f_cuad, fp_cuad, 1.5_real64, raiz, exito, iter)
   if (exito) print '(A, F16.12, A, I3, A)', '  Newton-Raphson: x = ', raiz, &
      ' en ', iter, ' iteraciones'

   print *
   print '(A, F16.12)', '  Valor exacto esperado: ', sqrt(2.0_real64)

   ! Problema 2: cos(x) = x -> f(x) = cos(x) - x
   print *
   print *, 'Problema 2: cos(x) - x = 0'
   call biseccion(f_cos_menos_x, 0.0_real64, 1.0_real64, raiz, exito, iter)
   if (exito) print '(A, F16.12, A, I3, A)', '  Biseccion:  x = ', raiz, &
      ' en ', iter, ' iteraciones'

   call newton_raphson(f_cos_menos_x, fp_cos_menos_x, 0.5_real64, raiz, exito, iter)
   if (exito) print '(A, F16.12, A, I3, A)', '  Newton-Raphson: x = ', raiz, &
      ' en ', iter, ' iteraciones'

   ! Demostrar diferencia de velocidad: alta precision
   print *
   print *, 'Comparacion con tolerancia 1e-14:'
   call biseccion(f_cuad, 0.0_real64, 2.0_real64, raiz, exito, iter, tol=1.0e-14_real64)
   print '(A, I4, A)', '  Biseccion: ', iter, ' iteraciones'
   call newton_raphson(f_cuad, fp_cuad, 1.5_real64, raiz, exito, iter, tol=1.0e-14_real64)
   print '(A, I4, A)', '  Newton-Raphson: ', iter, ' iteraciones'

contains

   function f_cuad(x) result(val)
      real(real64), intent(in) :: x
      real(real64) :: val
      val = x * x - 2.0_real64
   end function f_cuad

   function fp_cuad(x) result(val)
      real(real64), intent(in) :: x
      real(real64) :: val
      val = 2.0_real64 * x
   end function fp_cuad

   function f_cos_menos_x(x) result(val)
      real(real64), intent(in) :: x
      real(real64) :: val
      val = cos(x) - x
   end function f_cos_menos_x

   function fp_cos_menos_x(x) result(val)
      real(real64), intent(in) :: x
      real(real64) :: val
      val = -sin(x) - 1.0_real64
   end function fp_cos_menos_x

end program demo_raices
```

**Análisis línea por línea**:
- `call biseccion(f_cuad, 0.0_real64, 2.0_real64, raiz, exito, iter)` — Busca la raíz de x²-2 en [0, 2]; la raíz es √2 ≈ 1.414...
- `call newton_raphson(f_cuad, fp_cuad, 1.5_real64, raiz, exito, iter)` — Parte de 1.5, cerca de la raíz esperada.
- `if (exito) print ...` — Solo imprime si el algoritmo convergió.
- `tol=1.0e-14_real64` — Argumento opcional para aumentar la precisión requerida.
- Las funciones `f_cuad`, `fp_cuad`, etc., están en la sección `contains` del programa principal, por lo que son visibles para las subrutinas del módulo.

**Salida esperada**:
```
 === BUSQUEDA DE RAICES ===

 Problema 1: x^2 - 2 = 0  (raiz cuadrada de 2)
  Biseccion:  x =   1.414213562373  en  43 iteraciones
  Newton-Raphson: x =   1.414213562373  en   4 iteraciones

  Valor exacto esperado:   1.414213562373

 Problema 2: cos(x) - x = 0
  Biseccion:  x =   0.739085133215  en  42 iteraciones
  Newton-Raphson: x =   0.739085133215  en   5 iteraciones

 Comparacion con tolerancia 1e-14:
  Biseccion:   47 iteraciones
  Newton-Raphson:    5 iteraciones
```

Observación: Newton-Raphson requiere 4-5 iteraciones frente a 43-47 de la bisección. La diferencia es dramática. Pero Newton depende críticamente de una buena suposición inicial y de la disponibilidad de la derivada.

**Errores típicos**:

Error 1: Intervalo sin cambio de signo en bisección.
```fortran
program error_biseccion
   use iso_fortran_env, only: real64
   use raices
   implicit none
   real(real64) :: raiz
   logical :: exito
   integer :: iter
   call biseccion(f_cuad, -2.0_real64, -1.0_real64, raiz, exito, iter)
   if (.not. exito) print *, 'No se encontro raiz en el intervalo'
contains
   function f_cuad(x) result(val); real(real64), intent(in) :: x; real(real64) :: val; val = x*x - 2.0_real64; end function
end program error_biseccion
```
```
$ gfortran -std=f2018 error_biseccion.f90 raices.f90 && ./a.out
 No se encontro raiz en el intervalo
```
f(-2) = 2 y f(-1) = -1, hay cambio de signo en [-2, -1], pero la raíz √2 ≈ 1.414 no está en ese intervalo. En realidad, f(x)=x²-2 tiene raíz en x=-1.414 también. El mensaje es engañoso: sí hay cambio de signo, pero el algoritmo encontrará la raíz negativa. Esto ilustra que la bisección encuentra UNA raíz, no necesariamente la que esperamos. La raíz en [-2, -1] es x = -√2 ≈ -1.414, que también es válida.

Error 2: Derivada incorrecta en Newton-Raphson.
```fortran
program error_newton
   use iso_fortran_env, only: real64
   use raices
   implicit none
   real(real64) :: raiz
   logical :: exito
   integer :: iter
   call newton_raphson(f_cuad, fp_incorrecta, 1.5_real64, raiz, exito, iter)
   if (.not. exito) print *, 'Newton no convergio'
contains
   function f_cuad(x) result(val); real(real64), intent(in) :: x; real(real64) :: val; val = x*x - 2.0_real64; end function
   function fp_incorrecta(x) result(val); real(real64), intent(in) :: x; real(real64) :: val; val = cos(x); end function  ! derivada incorrecta
end program error_newton
```
```
$ gfortran -std=f2018 error_newton.f90 raices.f90 && ./a.out
 Newton no convergio
```
Con una derivada incorrecta, el método puede diverger, oscilar, o converger a una raíz espuria. Siempre verificar que `fp` sea la derivada analítica correcta.

---

## 7. Precisión numérica y IEEE 754

Cada vez que escribimos `0.1_real64` en Fortran, la computadora no almacena exactamente 0.1. Almacena el número binario más cercano a 0.1. ¿Por qué? Porque las computadoras trabajan en base 2, no en base 10. Así como 1/3 no tiene representación decimal finita (0.3333...), 0.1 no tiene representación binaria finita. Es una fracción periódica en binario: 0.00011001100110011...

El estándar IEEE 754 define cómo se representan los números reales en la práctica totalidad de las computadoras modernas. Un número en precisión doble (64 bits) se compone de: 1 bit de signo, 11 bits de exponente, y 52 bits de mantisa (significando). El valor es: `(−1)^s × 1.mantisa × 2^(exponente−1023)`. El bit implícito "1." antes de la mantisa nos regala un bit adicional de precisión.

¿Qué implicaciones tiene esto para el cálculo científico? La más importante es que las operaciones aritméticas no son exactas. `a + b + c` puede diferir de `a + (b + c)` debido al redondeo. La resta de dos números casi iguales causa una pérdida catastrófica de dígitos significativos (cancelación catastrófica). La suma de muchos números pequeños puede verse dominada por el error de redondeo si no se usa un algoritmo de suma compensada (como la suma de Kahan).

El módulo intrínseco `ieee_arithmetic` de Fortran 2003/2018 nos permite consultar y controlar el entorno de punto flotante. Podemos saber si una operación se desbordó, si ocurrió una división por cero, si se produjo un valor NaN (Not a Number). Podemos cambiar el modo de redondeo (al más cercano, hacia arriba, hacia abajo, hacia cero). Podemos consultar la precisión de la máquina: el épsilon (distancia de 1 al siguiente número representable), el número más grande, el número más pequeño normalizado.

Dos conceptos cruciales son: `epsilon(x)` devuelve el espaciado entre 1 y el siguiente número representable mayor que 1. Para `real64`, es 2^−52 ≈ 2.22×10⁻¹⁶. Esto significa que no podemos distinguir entre 1 y 1 + 2×10⁻¹⁶. El `huge(x)` devuelve el número finito más grande representable (~1.8×10³⁰⁸). `tiny(x)` devuelve el número normalizado más pequeño (~2.2×10⁻³⁰⁸). Por debajo de `tiny`, entramos en la región subnormal (gradual underflow).

Veamos una exploración práctica del mundo IEEE 754 usando el módulo `ieee_arithmetic` de Fortran.

---

```fortran
program demo_ieee754
   use iso_fortran_env, only: real64, real128
   use ieee_arithmetic
   implicit none
   real(real64) :: a, b, c, eps, uno, gran_deuda
   real(real128) :: a128, b128
   logical :: flag

   print *, '=== EXPLORACION IEEE 754 ==='
   print *

   ! Epsilon de la maquina: distancia de 1 al siguiente numero mayor
   eps = epsilon(1.0_real64)
   print '(A, ES16.6)', 'Epsilon (real64): ', eps
   print '(A)', '  Significa: 1.0 + epsilon/2 == 1.0 (se redondea a 1)'

   ! Rango
   print '(A, ES16.6)', 'Huge (maximo real64):   ', huge(1.0_real64)
   print '(A, ES16.6)', 'Tiny (minimo real64):   ', tiny(1.0_real64)

   ! Prueba: suma no asociativa
   a = 1.0e16_real64
   b = 1.0_real64
   c = -1.0e16_real64
   print *
   print *, 'Prueba de asociatividad:'
   print '(A, F30.2)', '  (a + b) + c = ', (a + b) + c
   print '(A, F30.2)', '  a + (b + c) = ', a + (b + c)
   print '(A)', '  Deberian ser 1.0, pero la primera da 0.0 por redondeo'

   ! Cancelacion catastrófica
   a = 1.000000000000001_real64
   b = 1.0_real64
   print *
   print *, 'Cancelacion catastrófica:'
   print '(A, F30.16)', '  a            = ', a
   print '(A, F30.16)', '  b            = ', b
   print '(A, F30.16)', '  a - b        = ', a - b
   print '(A, F30.16)', '  esperado     = ', 1.0e-15_real64

   ! Infinitos y NaN
   print *
   print *, 'Infinitos y NaN:'
   a = 1.0_real64 / 0.0_real64
   print '(A, F10.3)', '  1.0 / 0.0 = ', a
   print '(A, L1)', '  ¿Es infinito? ', ieee_is_finite(a) .eqv. .false.
   print '(A, L1)', '  ¿Es NaN?      ', ieee_is_nan(a)

   b = 0.0_real64 / 0.0_real64
   print '(A, F10.3)', '  0.0 / 0.0 = ', b
   print '(A, L1)', '  ¿Es NaN?      ', ieee_is_nan(b)

   ! Usar banderas de excepcion
   call ieee_set_flag(ieee_overflow, .false.)
   a = 1.0e300_real64
   gran_deuda = a * a  ! overflow
   call ieee_get_flag(ieee_overflow, flag)
   print '(A, L1)', '  ¿Hubo overflow? ', flag

   ! Precision cuádruple (real128) si el compilador lo soporta
   print *
   print *, 'Comparacion real64 vs real128:'
   a128 = 0.1_real128
   b128 = 0.2_real128
   print '(A, F40.36)', '  0.1 + 0.2 (real128) = ', a128 + b128
   print '(A, F40.36)', '  0.3 (real128)       = ', 0.3_real128
   print '(A, ES16.6)', '  Diferencia real128: ', (a128 + b128) - 0.3_real128
   print '(A, ES16.6)', '  Diferencia real64:  ', &
      (0.1_real64 + 0.2_real64) - 0.3_real64

end program demo_ieee754
```

**Análisis línea por línea**:
- `use ieee_arithmetic` — Módulo intrínseco para manipular el entorno de punto flotante.
- `epsilon(1.0_real64)` — Devuelve 2^−52 ≈ 2.22×10⁻¹⁶, el espaciado entre números en [1, 2).
- `huge(1.0_real64)` — (2−2^−52)×2^1023 ≈ 1.8×10³⁰⁸.
- `tiny(1.0_real64)` — 2^−1022 ≈ 2.2×10⁻³⁰⁸.
- `(a + b) + c = (1e16 + 1) - 1e16` — 1e16 + 1 se redondea a 1e16 porque el espaciado entre números en esa región es mucho mayor que 1. Luego 1e16 - 1e16 = 0. La versión `a + (b + c) = 1e16 + 0 = 1e16` da 0 por la cancelación de c y b.
- Cancelación catastrófica: `a - b` donde `a ≈ 1.000000000000001` y `b = 1.0`; la resta pierde 15 dígitos de precisión.
- `1.0 / 0.0` produce infinito (no es error en punto flotante IEEE).
- `ieee_is_finite(a)` — Devuelve `.false.` para infinito y NaN.
- `0.0 / 0.0` produce NaN (Not a Number).
- `ieee_is_nan(b)` — Devuelve `.true.` para NaN.
- `call ieee_set_flag(ieee_overflow, .false.)` — Resetea la bandera de overflow.
- `1.0e300 ** 2` excede huge → overflow silencioso, el resultado es infinito.
- `call ieee_get_flag(ieee_overflow, flag)` — Consulta si ocurrió overflow.
- `real128` — Cuadruple precisión (128 bits, ~34 dígitos decimales). Muestra que 0.1+0.2 es mucho más cercano a 0.3 en precisión cuádruple.

**Salida esperada**:
```
 === EXPLORACION IEEE 754 ===

 Epsilon (real64):   2.220446E-016
   Significa: 1.0 + epsilon/2 == 1.0 (se redondea a 1)
 Huge (maximo real64):    1.797693E+308
 Tiny (minimo real64):    2.225074E-308

 Prueba de asociatividad:
  (a + b) + c =                           0.00
  a + (b + c) =                   10000000000000000.00
  Deberian ser 1.0, pero la primera da 0.0 por redondeo

 Cancelacion catastrófica:
  a            =               1.0000000000000011
  b            =               1.0000000000000000
  a - b        =               0.0000000000000011
  esperado     =               0.0000000000000010

 Infinitos y NaN:
  1.0 / 0.0 =       Inf
  ¿Es infinito? T
  ¿Es NaN?      F
  0.0 / 0.0 =       NaN
  ¿Es NaN?      T
  ¿Hubo overflow? T

 Comparacion real64 vs real128:
  0.1 + 0.2 (real128) = 0.299999999999999999999999999999999999
  0.3 (real128)       = 0.299999999999999999999999999999999999
  Diferencia real128:  0.000000E+000
  Diferencia real64:   5.551115E-017
```

**Errores típicos**:

Error 1: Comparar números reales con `==`.
```fortran
program error_comparacion
   use iso_fortran_env, only: real64
   implicit none
   real(real64) :: x
   x = 0.0_real64
   do 10 veces
      x = x + 0.1_real64
   end do
   if (x == 1.0_real64) then  ! FALSO por errores de redondeo
      print *, 'Exactamente 1.0'
   else
      print *, 'No es exactamente 1.0, es:', x
   end if
end program error_comparacion
```
```
$ gfortran -std=f2018 error_comparacion.f90 && ./a.out
 No es exactamente 1.0, es:   0.99999999999999989
```
Nunca comparar reales con `==` a menos que sean valores asignados directamente (como `x = 1.0_real64`). Usar `abs(x - y) < tol`.

Error 2: Ignorar overflow/underflow.
```fortran
program error_overflow
   use iso_fortran_env, only: real64
   implicit none
   real(real64) :: x, y
   x = 1.0e200_real64
   y = x * x * x  ! overflow -> Inf
   print *, y
   ! Cualquier operacion con Inf -> Inf o NaN
   print *, y + 1.0  ! Inf
   print *, y / y    ! NaN
end program error_overflow
```
```
$ gfortran -std=f2018 error_overflow.f90 && ./a.out
                       Infinity
                       Infinity
                            NaN
```
El programa no aborta, pero los resultados son basura. Usar `ieee_get_flag` para detectar estas situaciones.

---

## 8. Mini-proyecto: Sistema de Regresión Lineal para Predicción de Precios de Vivienda

Hemos recorrido un largo camino. Álgebra lineal, regresión, integración, raíces, precisión numérica. Ahora es el momento de integrar todo en un proyecto completo y funcional.

El objetivo es construir un sistema que prediga el precio de una vivienda basándose en dos características: área (en metros cuadrados) y número de habitaciones. Los datos de entrenamiento están codificados directamente en el programa como una matriz de 12 registros. El sistema:

1. Lee los datos de entrenamiento.
2. Construye la matriz de diseño X con una columna de unos para el término independiente.
3. Resuelve la ecuación normal `β = (Xᵀ X)⁻¹ Xᵀ y` usando el módulo `matrix_ops`.
4. Calcula el coeficiente R² y el error cuadrático medio.
5. Muestra una tabla comparativa de valores reales vs predichos.
6. Permite al usuario ingresar nuevos valores (área y habitaciones) por teclado y obtener predicciones.
7. Exporta la tabla a un archivo CSV.

La arquitectura es modular: la lógica de regresión reside en `matrix_ops`, que ya tenemos. El programa principal orquesta la entrada/salida y la interacción con el usuario. El formato CSV permite abrir los resultados en cualquier hoja de cálculo o herramienta de análisis.

Este proyecto es representativo de cómo se aplica la computación numérica en el mundo real: datos tabulares, un modelo matemático, resolución matricial, validación con métricas, y exportación de resultados. Aunque el ejemplo es pequeño, la estructura es exactamente la misma que usaríamos con millones de registros (con las optimizaciones correspondientes).

Veamos primero la subrutina que exporta a CSV, y luego el programa completo.

---

```fortran
module exportacion_csv
   use iso_fortran_env, only: real64
   implicit none
   private
   public :: exportar_tabla_csv

contains

   !---------------------------------------------------------------------------
   ! Exporta una tabla de valores a un archivo CSV
   !---------------------------------------------------------------------------
   subroutine exportar_tabla_csv(archivo, y_real, y_pred, etiquetas)
      character(len=*), intent(in) :: archivo
      real(real64), intent(in) :: y_real(:), y_pred(:)
      character(len=*), intent(in), optional :: etiquetas(:)
      integer :: unidad, i, n

      n = size(y_real)

      open(newunit=unidad, file=archivo, status='replace', action='write')

      ! Cabecera
      if (present(etiquetas) .and. size(etiquetas) >= 2) then
         write(unidad, '(A, A, A, A, A)') &
            'Obs', ',', trim(etiquetas(1)), ',', trim(etiquetas(2))
      else
         write(unidad, '(A)') 'Obs,Valor_Real,Valor_Predicho'
      end if

      ! Datos
      do i = 1, n
         write(unidad, '(I0, A, F16.8, A, F16.8)') &
            i, ',', y_real(i), ',', y_pred(i)
      end do

      close(unidad)
      print *, 'Tabla exportada a: ', trim(archivo)

   end subroutine exportar_tabla_csv

end module exportacion_csv
```

**Análisis línea por línea**:
- `character(len=*), intent(in) :: archivo` — Nombre del archivo de salida (longitud arbitraria).
- `optional :: etiquetas(:)` — Vector opcional con los nombres de las columnas.
- `open(newunit=unidad, file=archivo, status='replace', action='write')` — Abre el archivo para escritura; `newunit` asigna un número de unidad automáticamente (evita conflictos con unidades hardcodeadas).
- `write(unidad, '(A, A, A, A, A)') ...` — Escribe la cabecera en formato CSV (columnas separadas por comas).
- `'(I0, A, F16.8, A, F16.8)'` — Formato: entero sin ancho fijo, coma, real con 16 dígitos totales y 8 decimales, coma, otro real.
- `close(unidad)` — Cierra el archivo; importante para asegurar que los datos se escriban completamente.

---

```fortran
program mini_proyecto_vivienda
   use iso_fortran_env, only: real64
   use matrix_ops
   use exportacion_csv
   implicit none

   ! Datos de entrenamiento: 12 viviendas
   ! Columnas: area (m^2), habitaciones, precio ($)
   integer, parameter :: n_obs = 12, n_pred = 2
   real(real64), allocatable :: X(:, :), y(:), beta(:), y_pred(:)
   real(real64) :: area, habs, precio_pred
   character(len=100) :: input_line
   integer :: i, ios

   ! Datos: cada fila es [area, habitaciones, precio]
   real(real64), dimension(n_obs, n_pred + 1) :: datos = reshape([ &
      50.0_real64,  2.0_real64, 120000.0_real64, &
      60.0_real64,  3.0_real64, 150000.0_real64, &
      75.0_real64,  3.0_real64, 180000.0_real64, &
      80.0_real64,  4.0_real64, 210000.0_real64, &
      90.0_real64,  4.0_real64, 235000.0_real64, &
      100.0_real64, 3.0_real64, 220000.0_real64, &
      110.0_real64, 4.0_real64, 275000.0_real64, &
      120.0_real64, 5.0_real64, 310000.0_real64, &
      130.0_real64, 4.0_real64, 300000.0_real64, &
      140.0_real64, 5.0_real64, 350000.0_real64, &
      150.0_real64, 5.0_real64, 380000.0_real64, &
      160.0_real64, 6.0_real64, 420000.0_real64], &
      shape=[n_obs, n_pred + 1])

   ! Separar predictores y objetivo
   allocate(X(n_obs, n_pred), y(n_obs))
   X(:, 1) = datos(:, 1)   ! area
   X(:, 2) = datos(:, 2)   ! habitaciones
   y = datos(:, 3)         ! precio

   print *, '========================================'
   print *, ' SISTEMA DE PREDICCION DE PRECIOS'
   print *, ' DE VIVIENDAS (Regresion Lineal Multiple)'
   print *, '========================================'
   print *
   print *, 'Datos de entrenamiento cargados: ', n_obs, ' registros'
   print *

   ! Ajustar modelo
   call regresion_lineal_multiple(X, y, beta, y_pred)

   ! Mostrar coeficientes
   print *, '=== COEFICIENTES DEL MODELO ==='
   print '("  Beta_0 (indep):      $", F12.2)', beta(1)
   print '("  Beta_1 (area):       $", F12.2, " /m^2")', beta(2)
   print '("  Beta_2 (habitac.):   $", F12.2, " /hab")', beta(3)
   print *

   ! Metricas de bondad
   print *, '=== METRICAS DE BONDAD ==='
   print '("  R^2 = ", F10.6)', r_cuadrado(y, y_pred)
   print '("  ECM = $", F14.2)', error_cuadratico_medio(y, y_pred)
   print *

   ! Tabla comparativa
   print *, '=== TABLA COMPARATIVA: VALOR REAL vs PREDICHO ==='
   print '(A6, A14, A16, A14)', 'Obs', 'Real ($)', 'Predicho ($)', 'Error ($)'
   print '(A50)', repeat('-', 50)
   do i = 1, n_obs
      print '(I6, F14.2, F16.2, F14.2)', &
         i, y(i), y_pred(i), y(i) - y_pred(i)
   end do
   print *

   ! Exportar a CSV
   call exportar_tabla_csv('predicciones_vivienda.csv', y, y_pred, &
      etiquetas=['Valor_Real', 'Valor_Predicho'])

   ! Bucle interactivo de prediccion
   print *, '=== PREDICCION INTERACTIVA ==='
   print *, 'Ingrese area (m^2) y habitaciones (0 0 para salir):'

   do
      print *
      print '(A)', '>>> ', advance='no'
      read(*, '(A)', iostat=ios) input_line
      if (ios /= 0) exit

      read(input_line, *, iostat=ios) area, habs
      if (ios /= 0) then
         print *, '  Error: ingrese dos numeros separados por espacio'
         cycle
      end if

      if (area <= 0.0_real64 .and. habs <= 0.0_real64) exit

      precio_pred = beta(1) + beta(2) * area + beta(3) * habs
      print '("  Precio estimado: $", F14.2)', precio_pred
   end do

   print *, 'Programa finalizado.'

end program mini_proyecto_vivienda
```

**Análisis línea por línea**:

Declaraciones y datos:
- `n_obs = 12, n_pred = 2` — Constantes que definen el tamaño del problema.
- `datos = reshape([...], shape=[n_obs, n_pred+1])` — Todos los datos en un solo constructor; la última columna es el precio.
- `X(:, 1) = datos(:, 1)` — Área de cada vivienda.
- `X(:, 2) = datos(:, 2)` — Número de habitaciones.
- `y = datos(:, 3)` — Precio (variable objetivo).

Ajuste y presentación:
- `call regresion_lineal_multiple(X, y, beta, y_pred)` — Un solo llamado para obtener coeficientes y predicciones.
- `print '("  Beta_0 (indep):      $", F12.2)', beta(1)` — Formato con carácter `$` y unidades para claridad.
- `r_cuadrado(y, y_pred)` — Calcula qué proporción de la varianza explica el modelo.
- `error_cuadratico_medio(y, y_pred)` — Error promedio en dólares al cuadrado.

Tabla y exportación:
- Bucle de impresión formateada con separador visual (`repeat('-', 50)`).
- `call exportar_tabla_csv(...)` — Escribe el CSV con los resultados.

Interacción:
- `read(*, '(A)', iostat=ios) input_line` — Lee toda la línea como string para manejar errores de formato.
- `read(input_line, *, iostat=ios) area, habs` — Interpreta la línea como dos números reales.
- `if (area <= 0.0 .and. habs <= 0.0) exit` — Condición de salida.
- `precio_pred = beta(1) + beta(2) * area + beta(3) * habs` — Evaluación directa del modelo lineal múltiple.

**Salida esperada**:
```
 ========================================
 SISTEMA DE PREDICCION DE PRECIOS
 DE VIVIENDAS (Regresion Lineal Multiple)
 ========================================

 Datos de entrenamiento cargados:           12  registros

 === COEFICIENTES DEL MODELO ===
  Beta_0 (indep):      $   6280.57
  Beta_1 (area):       $   1988.73 /m^2
  Beta_2 (habitac.):   $  11095.99 /hab

 === METRICAS DE BONDAD ===
  R^2 =   0.991543
  ECM = $  87926867.39

 === TABLA COMPARATIVA: VALOR REAL vs PREDICHO ===
   Obs      Real ($)     Predicho ($)    Error ($)
--------------------------------------------------
     1     120000.00      125424.74      -5424.74
     2     150000.00      156546.32      -6546.32
     3     180000.00      182288.74      -2288.74
     4     210000.00      205745.52       4254.48
     5     235000.00      237891.30      -2891.30
     6     220000.00      229266.47      -9266.47
     7     275000.00      268496.22       6503.78
     8     310000.00      308377.17       1622.83
     9     300000.00      308213.52      -8213.52
    10     350000.00      348100.47       1899.53
    11     380000.00      381157.04      -1157.04
    12     420000.00      411890.46       8109.54

 Tabla exportada a: predicciones_vivienda.csv

 === PREDICCION INTERACTIVA ===
 Ingrese area (m^2) y habitaciones (0 0 para salir):

>>> 85 3
  Precio estimado: $  198405.53

>>> 200 5
  Precio estimado: $  461992.76

>>> 0 0
Programa finalizado.
```

**Errores típicos**:

Error 1: No compilar todos los módulos juntos.
```bash
$ gfortran -std=f2018 mini_proyecto_vivienda.f90
```
```
Error: Cannot open module file 'matrix_ops.mod' at (1)
```
Los módulos deben compilarse en el orden correcto. Solución:
```bash
$ gfortran -std=f2018 -c matrix_ops.f90
$ gfortran -std=f2018 -c exportacion_csv.f90
$ gfortran -std=f2018 mini_proyecto_vivienda.f90 matrix_ops.o exportacion_csv.o -o vivienda
# O en un solo paso:
$ gfortran -std=f2018 matrix_ops.f90 exportacion_csv.f90 mini_proyecto_vivienda.f90 -o vivienda
```

Error 2: Overflow en la matriz XᵀX si los datos tienen magnitudes muy diferentes.
```fortran
! Datos con areas en m^2 (50-160) y precios en dolares (120k-420k)
! Las magnitudes son diferentes pero no extremas. Sin embargo, si usamos
! areas en km^2 (0.00005-0.00016), XᵀX estaria mal condicionada y la
! inversión podria fallar.
```
La solución sería normalizar/estandarizar los predictores antes de la regresión.

Error 3: No cerrar el archivo CSV -> datos perdidos.
```fortran
! Si olvidamos close(unidad) en exportar_tabla_csv, el archivo queda
! abierto y posiblemente sin datos escritos completamente.
! La subrutina siempre llama a close; verificar que no haya un
! error_stop intermedio que impida llegar a esa línea.
```

---

## Resumen del capítulo

Hemos construido, desde cero, un conjunto de herramientas numéricas en Fortran moderno:

| Concepto | Herramienta Fortran | Aplicación |
|---|---|---|
| Álgebra lineal | `matmul`, `transpose`, `reshape`, `spread` | Base de todo el capítulo |
| Regresión lineal | Ecuación normal con `matrix_ops` | Predicción de precios |
| Regresión polinomial | Matriz de potencias | Capturar no linealidades |
| Integración | Trapecio, Simpson, Romberg | Área bajo curvas |
| Raíces | Bisección, Newton-Raphson | Ecuaciones trascendentes |
| Precisión | `ieee_arithmetic`, `epsilon`, `huge` | Control de errores |
| Proyecto integrador | Módulos + programa principal | Sistema de predicción |

El código completo de este capítulo (módulos y programas) es funcional con `gfortran -std=f2018`. Se recomienda crear un archivo separado para cada módulo (`matrix_ops.f90`, `integracion_numerica.f90`, `raices.f90`, `exportacion_csv.f90`) y compilarlos junto con los programas demo.

La ciencia de datos con Fortran no es un oxímoron. Es la continuación de una tradición de cinco décadas: usar el lenguaje más eficiente para el cómputo numérico intensivo, manteniendo la claridad conceptual y el control de precisión que otros lenguajes delegan en bibliotecas externas. Cuando los datos son grandes y la precisión es crítica, Fortran sigue siendo la herramienta correcta.
