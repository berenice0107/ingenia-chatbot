# IngenIA · Asistente Académico FIUBA

Chatbot web que responde consultas de estudiantes sobre el Calendario Académico
2026–2027 de la Facultad de Ingeniería (UBA): fechas de cursada, parciales,
integradoras, inscripciones, prórrogas y trámites en SIU Guaraní.

**Demo:** https://berenice0107.github.io/ingenia-chatbot/

---

## Cómo se usa

Se puede escribir la consulta en lenguaje natural o usar los botones de consulta
rápida del panel izquierdo. El panel también muestra las fechas clave del año con
cuentas regresivas calculadas contra la fecha actual.

No requiere instalación ni servidor: es un único archivo `index.html` que se abre
directamente en cualquier navegador.

---

## Arquitectura

Toda la aplicación vive en `index.html`, dividida en dos bloques bien separados:

### 1. Capa de datos

Contiene el contenido del asistente, sin lógica:

- `ENLACES` — URLs institucionales (SIU Guaraní, calendario, becas), centralizadas
  para no repetirlas. Las respuestas las referencian con marcadores `{SIU}`.
- `RESPUESTAS` — 8 respuestas de procedimiento (cómo/dónde hacer cada trámite).
- `CALENDARIO` — 10 grupos de períodos académicos con sus fechas en formato ISO.
- `CLAVES_EXACTAS` — 43 frases que identifican un tema sin ambigüedad.
- `EJEMPLOS` — 104 formas distintas de preguntar, repartidas en 18 temas.

Para agregar o corregir contenido se edita este bloque, sin tocar el motor.

### 2. Motor de respuestas

Recibe la consulta del usuario y decide qué contestar. **No es un motor de
palabras clave:** usa recuperación de información por similitud vectorial.

---

## Cómo funciona el motor

### Preprocesamiento

Cada texto (consultas y ejemplos) pasa por el mismo pipeline:

1. **Normalización** — minúsculas y eliminación de tildes por descomposición
   Unicode NFD, para que "inscripción" e "inscripcion" sean equivalentes.
2. **Tokenización** — se separa en palabras y se descartan las de menos de
   3 caracteres.
3. **Palabras vacías** — se eliminan términos sin poder discriminante
   ("que", "para", "necesito", "cuando"...).
4. **Lematización** — se unifican variantes de un mismo término. Un conjunto de
   reglas generales resuelve el plural (`parciales → parcial`) y los sustantivos
   derivados (`inscripciones → inscripcion`), y una tabla de raíces del dominio
   unifica las formas verbales frecuentes: "me anoto", "anotarme" y "anotarse"
   se reducen todas a `anotar`.

### Ponderación TF-IDF

Cada una de las 104 frases de ejemplo se convierte en un vector sobre un
vocabulario de 98 términos. El peso de un término *t* en un documento *d* es:

```
peso(t, d) = tf(t, d) × idf(t)

tf(t, d)   = frecuencia de t en d / cantidad de términos de d
idf(t)     = log( N / (1 + df(t)) ) + 1
```

donde *N* es la cantidad de documentos y *df(t)* la cantidad de documentos que
contienen *t*. La idea del **idf** es que un término que aparece en casi todos los
ejemplos (como "materia") aporta poco para distinguir un tema de otro, mientras
que uno raro (como "correlativa") es muy informativo. Cada vector se normaliza a
longitud 1.

### Similitud del coseno

La consulta del usuario se vectoriza igual y se compara contra todos los
documentos mediante el coseno del ángulo entre vectores:

```
similitud(q, d) = (q · d) / (‖q‖ × ‖d‖)
```

Como los vectores están normalizados, el cálculo se reduce al producto escalar.
El resultado va de 0 (ninguna palabra en común) a 1 (idénticos). Cada tema se
puntúa con su mejor frase de ejemplo.

### Cascada de resolución

El motor responde por niveles, de mayor a menor confianza:

| Nivel | Criterio | Comportamiento |
|-------|----------|----------------|
| 1 | Coincidencia exacta de frase clave | Responde directamente |
| 2 | Similitud del coseno ≥ 0,40 | Responde el tema mejor puntuado |
| 3 | Similitud ≥ 0,15 | Ofrece las 3 opciones más probables como botones |
| 4 | Por debajo del umbral | Mensaje de ayuda con los temas disponibles |

Dos mecanismos adicionales corrigen errores frecuentes:

- **Tipo de pregunta.** Se detecta si la consulta pregunta por una fecha
  ("cuándo", "hasta cuándo", "plazo") o por un procedimiento ("cómo", "dónde",
  "pasos"), y se prioriza el tipo de respuesta correspondiente. Esto desambigua
  *"cómo me inscribo a materias"* (instructivo del SIU) de
  *"hasta cuándo me inscribo a materias"* (fechas del calendario).
- **Cobertura de vocabulario.** Si el motor reconoce menos del 60 % de las
  palabras de la consulta, no responde con confianza aunque un término suelto
  puntúe alto: baja al nivel de sugerencias. Evita responder cualquier cosa ante
  preguntas fuera de tema.

### Fechas

Las fechas se guardan como texto ISO (`"2026-03-09"`) y se parsean manualmente
descomponiendo el string, porque `new Date("2026-03-09")` se interpreta como UTC
y en Argentina devolvería el día anterior. Los estados ("en curso", "faltan N
semanas", "ya finalizó") se calculan contra la fecha del sistema, así que el
panel se mantiene actualizado solo.

La tarjeta de fechas clave del panel izquierdo se genera desde `CALENDARIO`: antes
estaba escrita a mano y podía quedar desincronizada del motor.

---

## Evaluación

El proyecto incluye un set de 38 casos de prueba con su tema esperado. Para
correrlo, abrir la consola del navegador (F12) y escribir:

```javascript
IngenIA.evaluar()
```

Devuelve la precisión y una tabla con los casos que falla. También se puede
inspeccionar el ranking de una consulta puntual:

```javascript
IngenIA.rankear("hasta cuando me anoto a materias")
```

### Resultados

Comparación entre el motor anterior (coincidencia de substrings) y el actual,
sobre un conjunto de 58 consultas:

| | Motor anterior | Motor actual |
|---|---|---|
| Respuestas correctas | 34/58 (58,6 %) | 58/58 (100 %) |
| Consultas sin respuesta | 17 | 0 |

> **Nota metodológica:** los umbrales y el corpus de ejemplos fueron ajustados
> observando estos mismos casos, por lo que el 100 % está sobreestimado. Una
> medición limpia requiere un conjunto de consultas nuevo, redactado sin haber
> visto el comportamiento del motor.

---

## Limitaciones conocidas

- **Errores de tipeo.** La comparación es entre palabras completas: "parciles" no
  se asocia a "parciales". Se resolvería incorporando distancia de Levenshtein.
- **Sin contexto conversacional.** Cada consulta se resuelve de forma
  independiente; no se pueden hacer repreguntas ("¿y el del segundo cuatrimestre?").
- **Sin filtrado dentro de un tema.** Ante "finales de diciembre" se devuelven
  todos los períodos de mesas, sin acotar al pedido.
- **Datos cargados manualmente.** El calendario cubre 2026–2027 y debe
  actualizarse a mano cada año desde la fuente oficial.

---

## Evolución del proyecto

1. **Prototipo en Python** (Google Colab): diccionario de respuestas y
   coincidencia de palabras clave, ejecutado por consola.
2. **Primera versión web**: traducción del prototipo a JavaScript e interfaz de
   chat, con incorporación del calendario académico y las cuentas regresivas.
3. **Versión actual**: separación entre capa de datos y motor, y reemplazo de la
   coincidencia de palabras por recuperación TF-IDF con cascada de confianza.

El motor actual se ejecuta íntegramente en el navegador (JavaScript). El
prototipo en Python corresponde a la primera etapa y no forma parte de la
aplicación desplegada.

---

## Fuentes

- Calendario Académico FIUBA: https://www.fi.uba.ar/docentes/calendario-academico
- SIU Guaraní: https://guaraniautogestion.fi.uba.ar/g3w/

## Autores

Trabajo grupal para la materia _(completar: nombre de la materia y cátedra)_.

- _(completar: integrante 1)_
- _(completar: integrante 2)_
- _(completar: integrante 3)_
