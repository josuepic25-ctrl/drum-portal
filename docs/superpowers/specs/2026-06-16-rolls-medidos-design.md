# Rolls / Redobles Medidos — Design Spec

**Fecha:** 2026-06-16
**Archivo afectado:** `index.html` (portal de partituras, single-file)
**Alcance:** Sistema completo de la familia de rolls (sostenidos + medidos). Reemplaza el "roll" único aproximado actual por una biblioteca de tipos con sonido y notación correctos. Diseñado para ser forward-compatible con los tresillos de la Fase 2 sin reescribir el motor.

## Objetivo

Que el portal reproduzca y escriba correctamente **cualquier redoble de la familia de rolls** —tanto colocado manualmente desde el grid como transcrito por el tutor desde una foto— de modo que el usuario entienda exactamente cómo suena cada redoble escrito en una partitura.

La queja que origina este trabajo: el roll actual emite golpes con espaciado uniforme idéntico para todo, así que "todos suenan igual" y no como los rudimentos practicados. La solución es modelar el **número de golpes y su densidad** por tipo de roll.

## Principio arquitectónico (forward-compat con tresillos)

**El roll define cuántos golpes, no su velocidad absoluta.** El espaciado real se calcula en tiempo de emisión a partir de la **duración de la celda anfitriona**:

```
espaciado = duración_de_la_celda / nº_de_golpes
```

- Fase 1 (esta): las celdas del grid son rectas → el roll suena recto.
- Fase 2 (tresillos): cuando una celda viva dentro de un grupo de tresillo, el **mismo objeto roll, sin cambios en el motor**, sonará y se notará en tresillo. El caso "5-stroke en tresillos dentro de un 4/4" funciona automáticamente.

**Restricción de implementación:** la matemática del roll NUNCA debe hardcodear semicorcheas/fusas como unidad. Siempre deriva el espaciado de la duración de la celda. Esto es lo que protege la Fase 2.

Referencia musical (confirma la necesidad de subdivisión variable): los redobles medidos se tocan contra una subdivisión regular que puede ser recta o de tresillo; el 5-stroke encaja natural en semicorcheas y el 7-stroke en corcheas de tresillo. Las rayas de trémolo indican la subdivisión (1=corcheas, 2=semicorcheas, 3=fusas). Fuente: https://rhythmnotes.net/drum-rolls/

## Modelo de datos (retrocompatible)

Se separa el ornamento de gracia (flam/drag/ruff) del roll, que pasa a un campo propio en la celda:

```js
// _cells[m][c][i] = null | 'rest' | {
//   dur, durH,
//   art?:  'accent' | 'ghost',
//   orn?:  'flam' | 'drag' | 'ruff',          // gracia — SIN CAMBIOS
//   roll?: { type, accentRelease }            // NUEVO
// }
```

`roll.type`:
- Sostenidos: `'buzz'`, `'single'`, `'double'`, `'triple'`
- Medidos: `'m5'`, `'m6'`, `'m7'`, `'m9'`, `'m10'`, `'m11'`, `'m13'`, `'m15'`, `'m17'`

`roll.accentRelease`: `boolean` (default `true`) — acentúa el golpe de cierre de los medidos.

Reglas:
- El nº de golpes de un medido es intrínseco al tipo: `m5`→5, `m6`→6, `m7`→7, `m9`→9, `m10`→10, `m11`→11, `m13`→13, `m15`→15, `m17`→17.
- `roll` y `orn` son mutuamente excluyentes en la práctica (una nota es roll **o** lleva gracia), pero el modelo no lo prohíbe; la UI no permite activar ambos a la vez.
- `art` puede coexistir con `roll` (un roll puede ir acentuado globalmente además del acento de cierre).
- Ausencia de `roll` ⇒ nota normal (comportamiento idéntico al actual).

### Migración de datos antiguos

Las celdas guardadas con el formato viejo `orn:'roll'` (string) se migran al cargar (biblioteca/localStorage, importar JSON, enlaces `?load=`): se elimina `orn:'roll'` y se sustituye por `roll:{ type:'buzz', accentRelease:true }`. Esto preserva scores guardados antes de este cambio. La migración corre en el punto donde se hidratan las celdas (`setState`/parser de import).

## Estado nuevo en DrumGridModule

```js
let _curRoll = null;   // null | { type, accentRelease }   (selección activa del picker)
```

Análogo a `_curArt`/`_curOrn`: persiste entre colocaciones hasta que el usuario lo cambia. `accentRelease` arranca en `true`.

## UX — Selector de rolls

El botón único "Roll" del grupo Ornamento se reemplaza por un **botón que abre un desplegable** (popover anclado al botón). El desplegable contiene:

- **Sección "Sostenidos":** Buzz · Single · Double · Triple
- **Sección "Medidos":** 5 · 6 · 7 · 9 · 10 · 11 · 13 · 15 · 17
- **Toggle "Acento en cierre"** (checkbox, default activado) — controla `accentRelease` del roll a colocar.

Comportamiento:
- Clic en un tipo lo selecciona (`_curRoll = { type, accentRelease }`), marca el botón principal del roll como activo y muestra una etiqueta corta del tipo elegido (p. ej. "5"). Cierra el desplegable.
- Clic de nuevo en el tipo ya activo (o un botón "Ninguno") apaga el roll (`_curRoll = null`).
- Seleccionar un roll desactiva el ornamento de gracia activo (`_curOrn = null`) y viceversa, por exclusión mutua en la UI.
- La colocación usa la **duración actual del picker** como span del roll. Así se cumplen las dos formas de entrada aprobadas: el **nombre** fija el nº de golpes; la **duración** fija el span (= notación de trémolo por debajo).

### Edición de notas existentes

Si hay un roll activo en el picker y el usuario hace clic en una celda **ya ocupada**, se aplica/alterna ese roll sobre la nota existente (igual que el patrón actual de art/orn). Sin roll activo, el clic sobre celda ocupada mantiene el comportamiento actual.

### Responsive (teléfono)

El desplegable debe caber y ser usable en viewport ~380px: items con tamaño de toque ≥32px, en cuadrícula que envuelve, sin desborde horizontal. El popover no debe salirse de la pantalla por la derecha.

## Generación de sonido (expansión de golpes)

En `buildVoice`, para una celda con `roll`, en lugar de emitir un acorde único se emite una secuencia de golpes. Con `durX` = duración de la celda en unidades de 1/32:

- **Medidos `mN`** (N = nº de golpes del tipo):
  - Repartir N golpes uniformemente sobre `durX`: `hitSz = floor(durX / N)`, con el remanente distribuido (patrón de descomposición ya usado para el roll actual).
  - Si `hitSz < 1` (la celda es demasiado corta para N golpes), usar `hitSz = 1` y emitir `min(N, durX)` golpes (degradación honesta; el usuario alargará la duración).
  - Último golpe acentuado (`!accent!`) si `accentRelease`.
- **buzz:** golpes muy densos a baja velocidad. `hitSz = 1` (la unidad mínima de 1/32), llenando `durX`; velocity baja (mapear a `!p!` o capa de síntesis) para el efecto crush.
- **double:** diddles llenando la duración (redoble largo). Densidad base `hitSz = 2` (una semicorchea en L:1/32); si la celda es más corta que eso, `hitSz = 1`.
- **single:** la mitad de denso que `double` ⇒ `hitSz = 4` (corchea en L:1/32), con el mismo manejo de remanente; degrada a `hitSz = 2` si la celda es corta.
- **triple:** golpes agrupados de a tres llenando `durX`; densidad equivalente a `double` pero contando los golpes en grupos de tres (relevante sobre todo para el render visual de tresillo).

Las densidades de los sostenidos (`hitSz` de 1/2/4) son los valores base para celdas largas; en celdas cortas se aplica el mismo recorte que en los medidos (`hitSz` mínimo 1, golpes recortados). Todas derivan de `durX`, nunca de una subdivisión fija.

Todos derivan el espaciado de `durX` (nunca de una subdivisión fija), cumpliendo el principio de forward-compat.

## Notación visual (abcjs)

- Rayas de trémolo según la subdivisión efectiva del roll. Calcular slashes a partir de la relación `durX / N` mapeada a corchea/semicorchea/fusa → `!trem1!` / `!trem2!` / `!trem3!`.
- Para sostenidos: `double`/`buzz` → `!trem3!` (o `!trem2!`); `single` → `!trem1!`; `triple` → `!trem3!` con indicación de tresillo si aplica.
- El acento de cierre se dibuja con `!accent!` sobre el último golpe.
- **Riesgo/validación:** abcjs puede no renderizar `!tremN!` como rayas sobre la plica. Se valida en implementación con el workflow de preview (snapshot/screenshot). *Fallback:* el estilo de marca actual (`!///!`) o, como mínimo, el sonido correcto con una anotación de texto del tipo de roll sobre la nota.

## Formato JSON / shorthand

- El char `r` en `tracks` se mantiene como **roll rápido genérico** = `roll:{ type:'buzz', accentRelease:true }`.
- Para un roll específico se usa el objeto `mods` (lo que emite el tutor al transcribir una foto):

```json
{
  "name": "Ejemplo redobles",
  "format": "shorthand",
  "timeSig": "4/4",
  "subdiv": 8,
  "measures": 1,
  "tracks": { "SN": "x...r...", "KI": "x...x..." },
  "mods": {
    "SN": {
      "0": { "roll": { "type": "m5" } },
      "4": { "roll": { "type": "m7", "accentRelease": true } }
    }
  }
}
```

Reglas del parser:
- `mods[inst][col].roll` sobrescribe/añade el roll sobre esa celda (debe existir nota en esa columna).
- Validación: `type` desconocido ⇒ error claro indicando instrumento y columna.
- Tabla de códigos documentada en `SKILL.md`: `buzz, single, double, triple, m5, m6, m7, m9, m10, m11, m13, m15, m17`.

## Persistencia / compartir

`getState`/`setState` ya hacen deep-copy de `_cells`, por lo que biblioteca, exportar/importar JSON y enlaces `?load=` soportan el campo `roll` automáticamente. La migración de `orn:'roll'` antiguo corre en la hidratación. Verificar con un caso guardado→recargado.

## SKILL.md (tutor)

Actualizar la sección de shorthand:
- Añadir la tabla de tipos de roll y el uso de `mods[...].roll`.
- Instruir al tutor a identificar, desde una foto, el tipo de redoble (contar rayas de trémolo + nº de golpes / nombre del rudimento) y emitir el `type` correcto.

## Criterios de aceptación

1. Cada tipo de roll (sostenidos y los 9 medidos) se selecciona desde el desplegable y se coloca en el grid.
2. Al reproducir, redobles distintos suenan con **densidad distinta** (un `m5` claramente menos denso que un `m17`; buzz claramente un crush).
3. El golpe de cierre de un medido suena acentuado cuando `accentRelease` está activo, y plano cuando se desactiva.
4. La partitura muestra rayas de trémolo (o el fallback acordado) para cada roll.
5. Importar el JSON de ejemplo (`tracks` + `mods.roll`) reconstruye los tipos de roll exactos.
6. Guardar en biblioteca y recargar preserva los rolls; un score viejo con `orn:'roll'` se migra a `roll:{type:'buzz'}` sin romperse.
7. Scores/celdas sin `roll` siguen funcionando idénticos (retrocompatibilidad).
8. Desplegable usable en viewport ~380px sin desborde.
9. **Forward-compat:** la expansión de golpes deriva el espaciado de la duración de la celda (verificable colocando el mismo roll sobre duraciones distintas y comprobando que el nº de golpes se mantiene pero el espaciado cambia).

## Riesgos

- **Render de trémolo en abcjs (`!tremN!`):** principal incógnita visual. Mitigado con validación de preview + fallback definido.
- **Densidad/velocity de buzz:** lograr el crush convincente puede requerir control en la capa de síntesis, no solo en ABC. Decisión final con verificación de audio.
- **Celdas demasiado cortas para N golpes:** degradación definida (hitSz=1, golpes recortados).

Cada riesgo tiene verificación explícita en el plan vía el workflow de preview (snapshot/screenshot/audio).

## No-objetivos (esta fase)

- El grid permanece **recto**; los tresillos como subdivisión real del grid son Fase 2 (el modelo ya los admite vía el principio arquitectónico).
- No se modela sticking L/R (en una caja no afecta el sonido).
- No se modelan rudimentos que no son de la familia roll (paradiddles, flam taps, etc.): ya son representables como notas sueltas con sus articulaciones/gracias existentes.
