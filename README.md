# Manual de Fortran Moderno

Manual completo y pedagógico de **Fortran moderno** (95/2003/2008/2018/2023) aplicado a análisis de datos, ciencia de datos, generación de reportes, REST APIs y HPC.

> 🔗 **Abrir el manual**: comienza por el [`01-indice.md`](01-indice.md)

---

## Archivos del manual

| Archivo | Contenido |
|---|---|
| [`01-indice.md`](01-indice.md) | Índice general, stack tecnológico, guía de uso |
| [`02-fundamentos-de-fortran-moderno.md`](02-fundamentos-de-fortran-moderno.md) | Cap. 0 — Fundamentos: variables, arrays, control, módulos, I/O |
| [`03-analisis-de-datos.md`](03-analisis-de-datos.md) | Cap. 1 — Análisis de datos: CSV, estadística, filtrado, histogramas |
| [`04-ciencia-de-datos.md`](04-ciencia-de-datos.md) | Cap. 2 — Ciencia de datos: álgebra lineal, regresión, integración numérica |
| [`05-generacion-de-reportes.md`](05-generacion-de-reportes.md) | Cap. 3 — Reportes: formatos, LaTeX, Gnuplot, exportación |
| [`06-rest-api-con-fortran.md`](06-rest-api-con-fortran.md) | Cap. 4 — REST API: bind(C), HTTP, JSON, servidor |
| [`07-proyecto-final-integrador.md`](07-proyecto-final-integrador.md) | Cap. 5 — Proyecto integrador: dashboard económico completo |
| [`08-hpc-con-fortran.md`](08-hpc-con-fortran.md) | Cap. 6 — HPC: OpenMP, Coarrays, DO CONCURRENT, 3 proyectos |

## Stack tecnológico

- **Compilador**: `gfortran` 13+ (`-std=f2018`)
- **Paralelismo**: OpenMP 4.5+ (`-fopenmp`), Coarrays, DO CONCURRENT
- **Herramientas**: `make`, `cmake`, `curl`, `gnuplot`, LaTeX
- **SO objetivo**: GNU/Linux (Ubuntu 22.04+), WSL2, macOS

## Estilo pedagógico

Cada bloque de código en el manual sigue el mismo patrón:

1. **Pre-explicación conceptual** (7–10 párrafos, sin código)
2. **Código Fortran** con comentarios en español
3. **Análisis línea por línea**
4. **Salida esperada**
5. **Errores típicos** (código que falla + mensaje real + solución)
6. **Mini-proyecto** ejecutable

## Requisitos

```bash
# Ubuntu/Debian
sudo apt install gfortran make cmake libcurl4-openssl-dev gnuplot

# macOS
brew install gcc make cmake curl gnuplot
```

Verificar:

```bash
gfortran --version   # >= 13
```

## Licencia

Manual educativo. El código de los ejemplos es de dominio público.
