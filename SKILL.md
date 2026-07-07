---
name: create-technical-manual
description: >
  Crea un manual técnico completo siguiendo el patrón pedagógico del repositorio.
  Genera archivos .md con pre-explicación conceptual (7-10 párrafos), código comentado,
  análisis línea por línea, salida esperada, errores típicos con código que falla,
  y mini-proyectos ejecutables.
---

# Skill: Crear manual técnico pedagógico

Esta skill permite generar manuales técnicos completos siguiendo el patrón
pedagógico utilizado en los manuales de este repositorio.

## Cuándo usar esta skill

- "Crea un manual de [tema]"
- "Genera documentación para [tecnología]"
- "Escribe un tutorial de [framework/herramienta]"
- "Haz una guía completa de [lenguaje/tecnología]"

## Instrucciones

### 1. Investigación previa

Antes de escribir, investiga:
- ¿Qué problema resuelve la tecnología?
- ¿Cuál es su filosofía de diseño?
- ¿Qué alternativas existen y en qué se diferencia?
- ¿Cuáles son los errores más comunes al empezar?

### 2. Planificación de la estructura

Define las secciones del manual:

```
1. Introducción (filosofía, por qué existe, comparativas)
   - Subtemas con pre-explicación + código + análisis + errores
   - Mini-proyecto
2. Configuración/instalación
   - Dependencias + primer ejemplo funcional
   - Errores típicos de instalación
3. Conceptos fundamentales
4. Tema intermedio 1
5. Tema intermedio 2
6. Proyecto completo
7. Ejercicios
8. Soluciones
```

### 3. Escritura de cada sección

Para cada sección, sigue este orden:

**Paso 1: Pre-explicación** (7-10 párrafos)
- Explica la filosofía: por qué la tecnología funciona así
- Analogía del mundo real
- Preguntas retóricas
- SIN código aún

**Paso 2: Código**
- Bloque de código con comentarios en español
- El código debe ser funcional y ejecutable

**Paso 3: Análisis línea por línea**
- Cada línea se explica individualmente
- Formato: `- explicación de esa línea`

**Paso 4: Salida esperada**
- Bloque con el output exacto

**Paso 5: Errores típicos**
- Para cada error: código que falla + mensaje real + solución

**Paso 6: Mini-proyecto**
- Código completo + análisis + salida

### 4. Checklist de calidad

Antes de finalizar, verifica:

- [ ] ¿Cada bloque de código tiene 7-10 párrafos de pre-explicación?
- [ ] ¿El análisis es línea por línea (no un resumen)?
- [ ] ¿Hay al menos 3 errores típicos con código que falla + mensaje real?
- [ ] ¿Cada sección principal tiene un mini-proyecto?
- [ ] ¿La salida esperada se muestra exactamente como la vería el usuario?
- [ ] ¿Los bloques de código especifican el lenguaje?
- [ ] ¿Los comentarios en el código están en español?
- [ ] ¿El archivo comienza con H1 y tiene ToC?
- [ ] ¿No hay bloques de código sin cerrar?

### 5. Formato de archivo

```
# [Título del manual]

> [Descripción breve]

---

**Stack tecnológico**: [tecnologías]

---

## 1. [Sección]
...
```

### 6. Variables de personalización

| Variable | Descripción | Ejemplo |
|---|---|---|
| `[TEMA]` | Tema principal del manual | "Selenium con JavaScript" |
| `[STACK]` | Tecnologías y versiones | "Node.js 22 · selenium-webdriver 4" |
| `[LENGUAJE]` | Lenguaje de los ejemplos | "rust", "javascript", "python" |
| `[PROYECTO]` | Proyecto a construir | "Sistema de Inventarios" |

### 7. Ejemplo de sección correcta

```markdown
## 2. Configuración del proyecto

[7-10 párrafos de pre-explicación...]

### 2.1 Dependencias

```lenguaje
// código
```

**Análisis línea por línea**:
- `línea 1`: explicación...

**Salida esperada**:
```
output
```

### 2.2 Errores típicos

**Error 1: olvidar X**.

```lenguaje
// código que FALLA
```
Mensaje: `error: ...`
Solución: ...

### 2.3 Mini-proyecto

```lenguaje
// código
```
**Análisis**:
- ...
**Salida esperada**:
```
```
```
