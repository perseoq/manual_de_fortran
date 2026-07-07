# Manual de Fortran Moderno para Análisis de Datos, Ciencia de Datos, Reportes y REST API

> Una guía completa y pedagógica para dominar Fortran moderno (95/2003/2008/2018/2023) aplicado al análisis de datos numéricos, ciencia de datos, generación de reportes automatizados y construcción de APIs RESTful. Diseñado para principiantes absolutos en Fortran que ya tienen nociones básicas de programación.

---

**Stack tecnológico**:
- **Compilador**: `gfortran` 13+ (GCC) — estándares Fortran 95/2003/2008/2018/2023
- **Paralelismo**: OpenMP 4.5+ (`-fopenmp`), Coarrays Fortran 2008/2018, `DO CONCURRENT` (F2018), MPI (via `bind(C)`)
- **Sistema operativo**: GNU/Linux (Ubuntu 22.04+), macOS (Homebrew), Windows (WSL2 o mingw-w64)
- **Bibliotecas auxiliares**: `json-fortran` 1.2 (JSON), `fson` (JSON alternativo), `netcdf-fortran` 4.5+ (datos científicos), `gnuplot` 5.4+ (gráficos), `libcurl` 7.81+ (HTTP)
- **Herramientas**: `make` 4.3+, `cmake` 3.22+, `python3` 3.10+ (solo para posprocesamiento y validación), `perf`/`gprof` (perfilado)
- **Editor recomendado**: Visual Studio Code con extensión Modern Fortran (fortran-lang), Vim/Neovim con `vim-fortran`, o Emacs con `f90-mode`

---

## Estructura del manual

| Archivo | Capítulo | Descripción | Páginas estimadas |
|---|---|---|---|
| [`02-fundamentos-de-fortran-moderno.md`](02-fundamentos-de-fortran-moderno.md) | **0 — Fundamentos de Fortran Moderno** | Instalación, variables, tipos, arrays, control de flujo, funciones, módulos, subrutinas, I/O con formato | ~50 |
| [`03-analisis-de-datos.md`](03-analisis-de-datos.md) | **1 — Manipulación y Análisis de Datos** | Lectura CSV/TSV, estadística descriptiva, filtrado con `where`, datos faltantes, ventanas deslizantes | ~40 |
| [`04-ciencia-de-datos.md`](04-ciencia-de-datos.md) | **2 — Ciencia de Datos y Computación Numérica** | Álgebra lineal, regresión lineal/polinomial, integración, raíces, splines, IEEE 754, `ieee_arithmetic` | ~45 |
| [`05-generacion-de-reportes.md`](05-generacion-de-reportes.md) | **3 — Generación de Reportes** | Formatos `write`, tablas dinámicas, exportación CSV/JSON/LaTeX, integración con Gnuplot, pipelines | ~35 |
| [`06-rest-api-con-fortran.md`](06-rest-api-con-fortran.md) | **4 — REST API con Fortran** | `bind(C)`, libcurl (cliente HTTP), servidor HTTP básico, JSON con `json-fortran`, rutas y métodos | ~40 |
| [`07-proyecto-final-integrador.md`](07-proyecto-final-integrador.md) | **5 — Proyecto Final Integrador** | Sistema completo: ingesta CSV → análisis estadístico → reporte LaTeX → API REST → gráficos | ~30 |
| [`08-hpc-con-fortran.md`](08-hpc-con-fortran.md) | **6 — HPC con Fortran Moderno** | OpenMP, Coarrays, DO CONCURRENT, optimización, perfilado, 3 proyectos completos de paralelismo | ~50 |

**Total estimado**: ~290 páginas equivalentes

---

## Convenciones del manual

### Símbolos y formato

- `**negrita**` — Primera aparición de un término técnico
- `` `código` `` — Nombres de funciones, variables, comandos, fragmentos de código en línea
- > **Nota**: — Información importante, advertencias o aclaraciones pedagógicas
- `!!!peligro` — Error común que puede causar fallos silenciosos
- `!!!tip` — Consejo de productividad o buena práctica

### Bloques de código

Todos los bloques de código especifican el lenguaje. Los más comunes son:

```fortran
program ejemplo
  implicit none
  print *, "¡Hola, Fortran Moderno!"
end program ejemplo
```

También verás bloques en `bash`, `yaml`, `makefile`, `json`, `latex`, `text` y `python`.

### Compilación y ejecución

Cada bloque de código Fortran del manual se compila con:

```bash
gfortran -std=f2018 -Wall -Wextra -o programa archivo.f90
./programa
```

Para proyectos multi-archivo:

```bash
gfortran -std=f2018 -Wall -Wextra -c modulo.f90
gfortran -std=f2018 -Wall -Wextra modulo.o programa.f90 -o programa
./programa
```

> **Nota**: Usamos `-std=f2018` como base. Cuando una característica es específica de F2003, F2008 o F2023, se indica explícitamente.

---

## Requisitos previos

### Conocimientos
- Saber qué es un archivo, una terminal y un editor de texto
- Haber programado antes en cualquier lenguaje (Python, C, Java, etc.)
- No necesitas saber Fortran — el capítulo 0 te lleva de 0 a competente

### Instalación del compilador

**Ubuntu/Debian**:
```bash
sudo apt update
sudo apt install gfortran make cmake libcurl4-openssl-dev
```

**macOS**:
```bash
brew install gcc make cmake curl
```

**Windows (WSL2)**:
```bash
# Instala Ubuntu desde Microsoft Store, luego:
sudo apt update && sudo apt install gfortran make cmake libcurl4-openssl-dev
```

**Verificar instalación**:
```bash
gfortran --version
# Deberías ver algo como:
# GNU Fortran (Ubuntu 13.2.0-...) 13.2.0
```

---

## Cómo usar este manual

1. **Principiante absoluto**: Comienza por el capítulo 0 (`02-fundamentos`). No te saltes las pre-explicaciones conceptuales: están diseñadas para que entiendas el *por qué* antes del *cómo*.
2. **Programador con experiencia en otros lenguajes**: Puedes empezar en el capítulo 0 y acelerar, pero no te saltes los errores típicos — el sistema de tipos de Fortran es único.
3. **Profesional de datos**: Los capítulos 1 y 2 son tu foco principal. El capítulo 0 te dará base suficiente si ya programas.
4. **Ingeniero de software**: El capítulo 4 (REST API) te sorprenderá — Fortran no es lo primero que viene a la mente para APIs, pero con `bind(C)` es perfectamente viable.
5. **Científico computacional**: El capítulo 6 (HPC) es tu destino. OpenMP, Coarrays y DO CONCURRENT te darán el poder de paralelizar simulaciones sin salir de Fortran.
6. **Estudiante**: Haz todos los mini-proyectos. Cada uno está diseñado para consolidar los conceptos del capítulo.

### Flujo de aprendizaje recomendado

```
Cap. 0 → Cap. 1 → Cap. 2 → Cap. 3 → Cap. 4 → Cap. 5 → Cap. 6
  │         │         │         │         │         │         │
  ▼         ▼         ▼         ▼         ▼         ▼         ▼
Fundam.  Análisis  Ciencia   Reportes  REST API  Integrador   HPC
         de datos  de datos
```

---

## Licencia

Este manual se distribuye con fines educativos. El código de los ejemplos y proyectos es de dominio público. Puedes usarlo, modificarlo y redistribuirlo libremente.

---

> Siguiente capítulo: [02-fundamentos-de-fortran-moderno.md](02-fundamentos-de-fortran-moderno.md)
