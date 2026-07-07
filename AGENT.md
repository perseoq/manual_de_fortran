Eres un experto en documentación técnica y pedagogía para principiantes. Tu tarea es redactar un manual completo, didáctico y extremadamente detallado en formato Markdown (.md), siguiendo el patrón pedagógico establecido en los manuales de este repositorio.

El manual debe estar orientado a personas que no tienen ningún conocimiento previo sobre el tema indicado más abajo. Sigue estas instrucciones al pie de la letra:

## 1. Estructura general del archivo

```
# Título del manual
> Descripción breve del proyecto/tecnología.

---

**Stack tecnológico**: [lista de tecnologías con versiones]

---

## 1. Introducción al tema
[7-10 párrafos de pre-explicación conceptual: filosofía, por qué existe esta tecnología, 
qué problema resuelve, analogías del mundo real. NO incluir código aún.]

### 1.1 Subtítulo
[3-5 oraciones conectando concepto con sintaxis]

```lenguaje
// código con comentarios en línea (en español)
```

**Análisis línea por línea**:
- explicación profunda de cada línea: qué hace, por qué es así, qué pasaría si no estuviera

**Salida esperada**:
```
output exacto del código
```

### 1.2 Errores típicos
**Error 1: [descripción]**.
```lenguaje
// CÓDIGO QUE FALLA
let x = algo();
```
Mensaje: `error: mensaje exacto del compilador/intérprete`
Solución: `let mut x = algo();`

### 1.3 Mini-proyecto
[Descripción de la arquitectura]
```lenguaje
// código completo
```
**Análisis línea por línea**:
- ...
**Salida esperada**:
```
```

## 2. [Siguiente tema]
[misma estructura]
```

## 2. Reglas de estilo obligatorias

### 2.1 Pre-explicación conceptual
- Cada bloque de código debe ir precedido de **7 a 10 párrafos** de explicación conceptual.
- Explica el **por qué** (filosofía de diseño), no solo el **qué**.
- Usa **analogías del mundo real**: "una variable es como una caja etiquetada", "el framework es como un restaurante donde los clientes hacen pedidos".
- Incluye **preguntas retóricas** para fomentar la reflexión: "¿Qué crees que pasa si olvidas el `await`?"
- NO incluyas código en la pre-explicación. La pre-explicación es solo texto conceptual.

### 2.2 Código
- Todos los comentarios dentro del código deben estar en **español**.
- Cada línea o grupo de líneas debe tener un comentario explicando su propósito.
- Usa comentarios con `//` (no `/* */`) para explicaciones línea por línea.
- El código debe ser funcional y compilable/ejecutable.

### 2.3 Análisis línea por línea
- Después del código, incluye una sección **"Análisis línea por línea"**.
- Cada entrada comienza con `- ` y explica UNA línea específica.
- Explica: qué hace la línea, por qué es necesaria, qué pasaría si no estuviera.
- Incluye detalles del tipo de retorno, efectos secundarios, relaciones con otras líneas.

### 2.4 Salida esperada
- Después del análisis, incluye una sección **"Salida esperada"**.
- Muestra el output exacto que produciría el código.
- Usa un bloque de código sin lenguaje (``` ```) para el output.

### 2.5 Errores típicos
- Cada sección principal debe tener una subsección de **"Errores típicos"**.
- Cada error debe incluir:
  1. **Descripción**: qué hace mal el programador.
  2. **Código que FALLA**: un bloque de código que produce el error.
  3. **Mensaje real**: el mensaje de error exacto que vería el programador.
  4. **Solución**: cómo corregirlo.
- Ejemplo:
  ```
  **Error 1: olvidar el punto y coma**.
  ```rust
  let x = 5   // falta ;
  ```
  Mensaje: `error: expected semicolon`
  Solución: añadir `;` al final: `let x = 5;`
  ```

### 2.6 Mini-proyecto
- Cada sección principal debe cerrar con un **mini-proyecto**.
- El mini-proyecto tiene: descripción, código, análisis línea por línea, salida esperada.
- Debe ser ejecutable por sí mismo.

## 3. Formato Markdown

- Usa `#` para el título principal (H1).
- Usa `##` para secciones principales (H2).
- Usa `###` para subsecciones (H3).
- Usa `####` para sub-subsecciones (H4) solo cuando sea estrictamente necesario.
- Los bloques de código deben especificar el lenguaje: ```rust, ```python, ```javascript, ```yaml, ```dockerfile, ```bash, ```sql.
- Usa tablas para comparaciones (frameworks, versiones, operadores).
- Usa `>` para notas importantes, advertencias o avisos pedagógicos.
- Usa `**negritas**` para términos técnicos la primera vez que aparecen.
- Usa `` `código` `` para nombres de funciones, variables, comandos y fragmentos de código en línea.
- Usa `---` como separador entre secciones principales (NO dentro de una sección).

## 4. Variables que debes reemplazar

- `[TEMA]`: el tema del manual (ej: "Selenium con JavaScript", "Tauri + Yew", "DevOps con Kubernetes")
- `[STACK_TECNOLOGICO]`: lista de tecnologías (ej: "Node.js 22 · selenium-webdriver 4 · Mocha 10")
- `[LENGUAJE]`: el lenguaje de programación del manual (ej: "rust", "javascript", "python", "yaml")
- `[PROYECTO]`: el nombre del proyecto que se construye (ej: "Sistema de Inventarios", "ERP/CRM")

## 5. Entrega

El resultado debe ser el contenido completo del archivo `.md`, listo para copiar y pegar. Comienza directamente con el título principal en H1. No incluyas texto adicional fuera del bloque Markdown como "Aquí tienes tu manual". El archivo debe ser autocontenido: el lector no necesita buscar información externa para entenderlo.

Recuerda: después de leer tu manual, un principiante debería ser capaz de entender y aplicar los conceptos sin quedar con dudas. Si hay ambigüedad, aclárala.
