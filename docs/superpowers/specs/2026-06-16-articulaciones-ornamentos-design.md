# Articulaciones y Ornamentos de Batería — Design Spec (Fase 1)

**Fecha:** 2026-06-16
**Archivo afectado:** `index.html` (portal de partituras, single-file, ~1899 líneas)
**Alcance:** Fase 1 de 2. Esta fase añade articulaciones y ornamentos a notas individuales. Los tresillos/grupillos irregulares (Fase 2) se especifican aparte porque requieren rediseñar el motor de subdivisión del grid.

## Objetivo

Dar soporte —en el grid editable, en la partitura renderizada (abcjs), en la reproducción y en el formato JSON/shorthand— a las articulaciones y ornamentos más usados en partituras de batería: acento, ghost note, flam, drag, ruff y roll. El usuario debe poder colocarlos manualmente desde el panel y el tutor/transcripción de fotos debe poder generarlos vía JSON.

## No-objetivos (Fase 1)

- Tresillos y cualquier grupillo irregular (van en Fase 2).
- Cambios de signatura mixtos dentro de un compás.
- Notación de dinámicas globales (cresc., dim.).

## Modelo de datos

Se extiende el objeto de celda de forma retrocompatible. Las celdas existentes (`{ dur, durH }`) siguen siendo válidas; los campos nuevos son opcionales.

```js
// _cells[m][c][i] = null | 'rest' | {
//   dur, durH,
//   art?: 'accent' | 'ghost',                 // articulación: símbolo + velocity
//   orn?: 'flam' | 'drag' | 'ruff' | 'roll'   // ornamento: grace notes / trémolo
// }
```

Reglas:
- `art` y `orn` son independientes: una nota puede ser flam acentuado (`art:'accent', orn:'flam'`).
- `'accent'` y `'ghost'` son mutuamente excluyentes (un campo único `art`).
- Ausencia de `art`/`orn` ⇒ nota normal (comportamiento actual idéntico).

## Estado nuevo en DrumGridModule

```js
let _curArt = null;   // null | 'accent' | 'ghost'   (toggle latcheable)
let _curOrn = null;   // null | 'flam' | 'drag' | 'ruff' | 'roll'   (selector single-choice)
```

Análogos a `_isDotted`: persisten entre colocaciones hasta que el usuario los cambia.

## UX — Panel de selección

Dos grupos nuevos, debajo de la fila de duraciones existente, con tooltips explicativos en español.

### Grupo "Articulación" (toggles, tipo puntillo)
- Botón `>` — Acento. `title="Acento (nota más fuerte)"`.
- Botón `( )` — Ghost. `title="Ghost note (nota fantasma, más suave)"`.

Comportamiento: clic activa/desactiva. Activar `accent` desactiva `ghost` y viceversa (mutuamente excluyentes). Se combinan con el puntillo (`_isDotted` es independiente).

### Grupo "Ornamento" (selector de una sola opción)
- Botones: `Flam`, `Drag`, `Ruff`, `Roll`. Tooltips: `"Flam (una nota de gracia)"`, `"Drag (dos notas de gracia)"`, `"Ruff (tres notas de gracia)"`, `"Roll (redoble / trémolo)"`.

Comportamiento: clic en uno lo selecciona y deselecciona los demás. Clic en el ya activo lo apaga (`_curOrn = null`).

### Edición de notas existentes
Si hay un toggle de articulación u ornamento activo y el usuario hace clic en una **celda ya ocupada**, se aplica/alterna ese modificador sobre la nota existente en lugar de crear una nueva. Si no hay toggle activo, el clic sobre celda ocupada mantiene el comportamiento actual (borrar/togglear la nota).

### Estilo visual de botones activos
Reutiliza la clase `.active` existente (fondo `--accent`). Los botones nuevos usan la misma clase `.dur-btn` para heredar estilos, más un `id` propio para el handler.

### Responsive (teléfono)
Los grupos deben envolver (`flex-wrap`) y mantener tamaño de toque ≥32px. Sin filas que desborden horizontalmente en viewport ~380px.

## Generación ABC (en `buildVoice` / `_buildABC`)

El orden de emisión por evento es: **grace notes → decoraciones → acorde → longitud**.

```
{graces}!deco1!!deco2![notas]LEN
```

Mapeo:

| Modificador | Token ABC | Posición |
|---|---|---|
| accent | `!accent!` | decoración, antes del acorde |
| ghost | `!(.!` `!).!` | decoraciones, antes del acorde (paréntesis) |
| flam | `{/g}` (gracia de la nota principal) | grace, antes de todo |
| drag | `{gg}` (dos gracias) | grace |
| ruff | `{ggg}` (tres gracias) | grace |
| roll | `!///!` (trémolo en plica) | decoración |

Notas de implementación:
- La nota de gracia usa el pitch de la **nota principal del evento** (para batería, normalmente snare). En un acorde multi-instrumento, usar el pitch de la nota de menor índice de voz presente en el evento.
- Los **ghost notes requieren boilerplate de decoración** en el header ABC. Se tomará la definición exacta de la guía de Lou Montulli (decoraciones `(.` y `).` que dibujan paréntesis) y se insertará una sola vez en el header junto a los `%%percmap`. Validar el render con el workflow de preview; si abcjs no dibuja los paréntesis con la boilerplate, fallback: notehead entre paréntesis vía anotación de texto, o como mínimo el efecto de volumen suave en reproducción.
- `!///!` (roll) es el mapeo primario; si abcjs no lo renderiza como trémolo sobre la plica, fallback a `!/!`/`!//!` o a la decoración `!trem3!`. Se valida en implementación.
- Las decoraciones de articulación se aplican al **acorde completo** del evento (van antes del `[`). Esto es correcto para charts de batería donde el acento marca el golpe entero.

## Reproducción (sonido)

abcjs synth respeta dinámicas vía decoraciones de volumen. Estrategia:
- `accent` ⇒ velocity alta (mapear a `!f!` o subir volumen de la nota).
- `ghost` ⇒ velocity baja (mapear a `!p!`/`!pp!` o bajar volumen) — debe sonar más suave.
- `flam/drag/ruff` ⇒ abcjs reproduce las grace notes automáticamente.
- `roll` ⇒ best-effort; prioridad es el símbolo visual.

Si las decoraciones de volumen interfieren con el render visual, el volumen se controla en la capa de síntesis (no en el ABC). Decisión final durante implementación con verificación de audio.

## Formato JSON / shorthand

Se amplía el vocabulario de caracteres por posición en `tracks`:

```
.  silencio       x  nota normal
X  acento         g  ghost
f  flam           d  drag        r  roll
```

(`ruff` no tiene char dedicado por baja frecuencia; se accede vía `mods`.)

Para combinaciones (ej. flam acentuado) o ruff, un objeto opcional `mods` por instrumento, indexado por columna:

```json
{
  "name": "Ejemplo",
  "format": "shorthand",
  "timeSig": "4/4",
  "subdiv": 8,
  "measures": 1,
  "tracks": { "SN": "g.X.x.f.", "KI": "x...x..." },
  "mods": { "SN": { "0": { "art": "accent", "orn": "flam" }, "6": { "orn": "ruff" } } }
}
```

Reglas del parser:
- Cada char de `tracks` se traduce a `{ dur:1, durH:2, art?, orn? }` según la tabla.
- Si `mods[inst][col]` existe, sobrescribe/añade `art`/`orn` sobre esa celda (debe haber nota en esa columna).
- Validación: caracteres desconocidos ⇒ error claro indicando posición e instrumento.

## Persistencia / compartir

Sin cambios. `getState`/`setState` ya hacen deep-copy de `_cells`, por lo que biblioteca (localStorage), exportar/importar JSON y enlaces `?load=` soportan los modificadores automáticamente. Verificar con un caso guardado→recargado.

## Criterios de aceptación

1. Colocar nota con cada toggle activo produce el símbolo correcto en la partitura (acento, paréntesis ghost, flam, drag, ruff, roll).
2. Acento + flam combinados renderizan ambos en la misma nota.
3. Clic en nota existente con toggle activo aplica/quita el modificador.
4. Ghost suena más suave y acento más fuerte en reproducción.
5. Importar el JSON de ejemplo (con `tracks` + `mods`) reconstruye exactamente esos modificadores.
6. Guardar en biblioteca y recargar preserva los modificadores.
7. Celdas/partituras antiguas sin modificadores siguen funcionando idénticas (retrocompatibilidad).
8. Panel usable en viewport de teléfono (~380px) sin desborde horizontal.

## Riesgos

- **Render de ghost notes en abcjs:** principal incógnita. Mitigado con boilerplate de Montulli + fallback + verificación visual.
- **Roll/trémolo:** mapeo a validar; fallback definido.
- **Volumen de reproducción:** abcjs puede no respetar dinámicas por nota; fallback a capa de síntesis.

Cada riesgo tiene verificación explícita en el plan vía el workflow de preview (snapshot/screenshot/audio).
