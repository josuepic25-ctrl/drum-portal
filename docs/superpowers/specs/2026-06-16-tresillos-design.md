# Tresillos / Subdivisión Mixta por Beat — Design Spec

**Fecha:** 2026-06-16
**Archivo afectado:** `index.html` (portal de partituras, single-file) + `SKILL.md` (tutor)
**Alcance:** Soporte de subdivisiones en tresillo conviviendo con notas rectas **dentro del mismo compás**, con granularidad por beat. Modo tresillo como modo de escritura (no transforma lo ya escrito). Forward-compat con el sistema de rolls de la Fase 1 (los rolls funcionan dentro de tresillos sin tocar el motor).

## Objetivo

Que el portal escriba, muestre y reproduzca correctamente compases que mezclan beats rectos y beats en tresillo —tanto colocados manualmente desde el grid como transcritos por el tutor desde una foto—. El caso de referencia es el **Solo No. 7** del repertorio de Linear Drumming / American Drummer: compases en 2/4 con beats de semicorcheas rectas junto a beats en tresillo, más rolls medidos (5/7/9 golpes) y acentos.

La limitación que origina este trabajo: hoy el grid es una rejilla rectangular uniforme (un único `_subdiv` para todo el compás) y el motor ABC usa enteros de 1/32, por lo que los tresillos no son representables.

## Principio arquitectónico

**La subdivisión es una propiedad del beat, no del compás.** Cada beat decide independientemente si es recto o tresillo. Esto permite mezclar libremente dentro de un compás (beat 1 recto + beat 2 tresillo) y refleja exactamente cómo se ve la notación real de batería.

**Las celdas internas no cambian.** Una celda dentro de un beat tresillo es idéntica a una celda recta: conserva `dur`, `durH`, `art`, `orn`, `roll`. El motor de rolls de la Fase 1 (`_expandRoll`, que deriva el espaciado de `durX`) **no se toca**: dentro de un tuplet, los golpes del roll se escalan automáticamente por el corchete. Esto cumple la promesa forward-compat que se diseñó en la Fase 1.

## Modelo de datos

Cada compás deja de ser una matriz rectangular `cols × instrumentos` y pasa a ser una lista de **grupos de beat**:

```js
// _cells[m] = [ beatGroup, beatGroup, ... ]      (un grupo por beat del compás)
//
// beatGroup = {
//   kind:  'straight' | 'triplet',
//   div:   number,        // nº de celdas del beat
//   cells: Cell[][]       // cells[colDelBeat][instrumento] = Cell
// }
//
// Cell = null | 'rest' | {
//   dur, durH,
//   art?:  'accent' | 'ghost',
//   orn?:  'flam' | 'drag' | 'ruff',
//   roll?: { type, accentRelease }     // SIN CAMBIOS respecto a Fase 1
// }
```

Reglas de subdivisión:

- **Nº de beats por compás** = `timeSig.num` (cada beat = una unidad del denominador). Esto es consistente con el conteo de columnas actual del portal: `beatsPorCompás = colsPerMeasure / colsPerBeat`, donde `colsPerBeat = subdiv / den` y `colsPerMeasure = subdiv × num/den`, cuyo cociente es exactamente `num`. Para 2/4 → 2 beats; 4/4 → 4 beats; 3/4 → 3 beats. (Los compases compuestos como 6/8 quedan fuera de foco en esta fase; ver No-objetivos.)
- **`div` de un beat recto** = `colsPerBeat = subdiv / den`, que con los compases /4 da: ×8 (eighth) → 2 celdas/beat; ×16 (sixteenth) → 4 celdas/beat.
- **`div` de un beat tresillo**: 3 celdas si el selector global es ×8 (eighth-note triplet); 6 celdas si es ×16 (sixteenth-note triplet / sextuplet).
- La densidad del tresillo (3 vs 6) se deriva del selector ×8/×16 existente — **no se añade control nuevo** para esto.

### Estructura interna de `cells`

Mantener `cells[col][instrumento]` (mismo layout columna-mayor que hoy) para minimizar cambios en render y emisión. La diferencia es que el número de columnas (`div`) varía por beat en vez de ser fijo por compás.

## Estado nuevo en DrumGridModule

```js
let _tripletMode = false;   // modo de escritura activo (toggle del botón Tresillo)
```

Análogo a los toggles existentes. Cuando está activo, los beats vacíos se muestran en tresillo y la primera nota colocada en un beat fija su `kind` a `'triplet'`.

## UX — Modo tresillo como modo de escritura

- **Botón toggle "Tresillo"** (con etiqueta "3") junto al selector de subdivisión ×8/×16.
- **Beats ya escritos conservan su subdivisión.** Activar/desactivar el modo nunca transforma contenido existente.
- **Beats vacíos** (todas sus celdas `null`) se renderizan según `_tripletMode`: off → `div` recto del selector; on → `div` de tresillo (3 o 6), con un corchete/etiqueta "3" encima del beat.
- **Fijación:** al colocar la primera nota en un beat vacío, ese beat queda fijado al `kind` correspondiente al modo activo en ese momento.
- **Reconversión de un beat ya escrito:** clic en la **etiqueta del beat** (la cabecera de columna que muestra el número de beat) alterna su `kind` recto↔tresillo. Si el beat tiene notas, pide confirmación (`confirm()`), porque al cambiar `div` las celdas no mapean 1:1 y se vacían.

### Responsive (teléfono)

El grid con beats de distinto ancho debe seguir cabiendo en viewport ~380px sin desborde horizontal. Los beats tresillo (3/6 celdas) y rectos (2/4 celdas) comparten fila; las celdas se ajustan en ancho para que el compás completo entre. Tamaño de toque de celda ≥28px.

## Emisión ABC (sonido + notación)

En `_buildABC`, recorrer cada compás **beat por beat**:

- **Beat recto:** emitir las celdas como hoy (sin envoltorio de tuplet).
- **Beat tresillo:** envolver las notas del beat en un tuplet ABC **explícito** `(p:q:r`:
  - eighth-note triplet (`div=3`): `(3:2:r`
  - sextuplet (`div=6`): `(6:4:r`
  - **`r` = número real de eventos-nota ABC emitidos dentro del beat**, contando los golpes que un `roll` expande. Calcular `r` recorriendo las celdas del beat y sumando: 1 por celda con nota simple, 1 por silencio, y `sizes.length` (resultado de `_expandRoll`) por celda con roll. Las notas de gracia (flam/drag/ruff) **no** cuentan para `r` (ABC las trata como adornos fuera del conteo del tuplet).
- El valor de duración de cada celda (face value en unidades L:1/32) se mantiene igual que en un beat recto del mismo `div` equivalente; el corchete del tuplet hace el escalado temporal. Para `div=3`, cada celda vale lo que una corchea recta (face value de eighth); para `div=6`, lo que una semicorchea.
- **Integración con rolls:** la celda con `roll` se expande con `_expandRoll(durX, type)` **sin cambios**. Los golpes resultantes quedan dentro del tuplet y se escalan con él. Verificar que `r` los incluye.

### Riesgo/validación de render

abcjs debe dibujar el corchete de tresillo "3" sobre las notas. Se valida en implementación con el workflow de preview (snapshot/screenshot). *Fallback:* si el corchete no aparece, mantener al menos el sonido correcto (el escalado temporal del tuplet) y una anotación de texto "3" sobre el beat.

## Formato JSON / shorthand

Se extiende el formato shorthand para marcar la subdivisión por beat. Las pistas (`tracks`) siguen siendo cadenas de caracteres, pero ahora se segmentan **por beat** con el separador `|`:

```json
{
  "name": "Solo 7 — compás de ejemplo",
  "format": "shorthand",
  "timeSig": "2/4",
  "subdiv": 16,
  "measures": 1,
  "beats": { "0": "straight", "1": "triplet" },
  "tracks": {
    "SN": "X..x|Xxx"
  },
  "mods": {
    "SN": { "0": { "art": "accent" } }
  }
}
```

Reglas del parser:

- **`beats`**: objeto `{ "<índiceGlobalDeBeat>": "straight" | "triplet" }`. Índice global = posición del beat contando desde el inicio del primer compás (0-based, atraviesa compases). Beats no listados ⇒ `straight` por defecto.
- **Separador `|`** en cada cadena de `tracks` separa los segmentos de cada beat. El nº de segmentos debe igualar el nº total de beats (`measures × beatsPorCompás`). La longitud de cada segmento debe igualar el `div` de ese beat (recto: 2 o 4 según `subdiv`; tresillo: 3 o 6 según `subdiv`).
- **Retrocompatibilidad del shorthand:** si una cadena **no contiene `|`**, se interpreta con el comportamiento actual (rectangular, todo recto, longitud = `subdiv × num/den × ...`). Así los grooves viejos del tutor siguen funcionando sin cambios.
- **`mods[inst][col]`**: `col` es el índice **global de columna** (atravesando beats y compases, en orden de emisión), igual que hoy pero contando las columnas reales del grid mixto. Documentar claramente en SKILL.md cómo contar columnas cuando hay beats de distinto `div`.
- **Validación:** nº de segmentos `|` ≠ nº de beats ⇒ error claro indicando la pista. Longitud de segmento ≠ `div` esperado ⇒ error indicando beat y pista. Valor de `beats` distinto de `straight`/`triplet` ⇒ error.

## Migración de datos antiguos

Los scores guardados con el formato rectangular (`_cells[m][col][inst]`) se migran al hidratar (biblioteca/localStorage, importar JSON, enlaces `?load=`):

- Cada compás rectangular se parte en beats `straight`. Para un compás con `cols` columnas y `beatsPorCompás` beats, cada beat recibe `cols / beatsPorCompás` columnas consecutivas (= `div` recto), copiando las celdas correspondientes.
- La migración corre en el punto donde se hidratan las celdas (`setState` / parser de import), junto a la migración de `orn:'roll'` ya existente de la Fase 1.
- Detección: si `_cells[m]` es un array cuyo primer elemento **no** tiene la forma `{ kind, div, cells }`, es formato viejo ⇒ migrar.

## Persistencia / compartir

`getState`/`setState` ya hacen deep-copy de `_cells`, por lo que biblioteca, exportar/importar JSON y enlaces `?load=` soportan la estructura de grupos-por-beat automáticamente. La migración del formato rectangular corre en la hidratación. Verificar con un caso guardado→recargado y con un score viejo→migrado.

## SKILL.md (tutor)

Actualizar la sección de shorthand:

- Documentar el campo `beats` (subdivisión por beat) y el separador `|` en las pistas.
- Documentar la regla de retrocompatibilidad (cadenas sin `|` = rectas).
- Guía de transcripción desde foto: identificar beats en tresillo por el **corchete "3"** sobre las notas, marcarlos en `beats`, y segmentar las pistas con `|`. Combinar con la identificación de rolls (corchete de trémolo + "N str") ya documentada en la Fase 1.
- Ejemplo completo basado en un compás del Solo No. 7 (2/4, beat recto + beat tresillo, con un acento).

## Criterios de aceptación

1. Un compás puede contener un beat recto y un beat tresillo simultáneamente; ambos se muestran con su subdivisión correcta (recto con celdas iguales, tresillo con corchete "3").
2. Activar el modo tresillo y escribir notas crea beats tresillo; desactivarlo y escribir crea beats rectos; **lo ya escrito no se transforma**.
3. Al reproducir, un beat tresillo suena con 3 (o 6) notas en el tiempo de un beat, claramente distinto de un beat recto.
4. Un roll colocado dentro de un beat tresillo suena con su densidad correcta **escalada al tresillo** (forward-compat de Fase 1 verificada), y el `r` del tuplet incluye los golpes del roll.
5. La partitura dibuja el corchete de tresillo (o el fallback acordado) sobre cada beat tresillo.
6. Importar el JSON de ejemplo (con `beats` + `|`) reconstruye la mezcla recto/tresillo exacta.
7. Un groove viejo del tutor (cadenas sin `|`, sin `beats`) se importa idéntico al comportamiento actual (retrocompat del shorthand).
8. Guardar en biblioteca y recargar preserva la mezcla; un score viejo rectangular se migra a beats `straight` sin romperse.
9. Reconvertir un beat recto↔tresillo desde su etiqueta funciona, con confirmación si tiene notas.
10. El grid mixto cabe en viewport ~380px sin desborde horizontal.

## Riesgos

- **Render del corchete de tresillo en abcjs:** principal incógnita visual. Mitigado con validación de preview + fallback (sonido correcto + anotación "3").
- **Conteo de `r` con rolls dentro de tresillos:** si `r` no incluye los golpes del roll, el corchete cubre mal las notas. Verificación explícita en el plan.
- **Refactor del modelo de celdas:** toca `_initCells`, `_buildABC`, `_renderTable`, handler de clic, `getState`/`setState`, parser shorthand y migración. Riesgo de regresión en el camino recto. Mitigado con verificación de que los grooves rectos existentes siguen idénticos (criterio 7) y TDD por tarea.
- **Anchos de celda mixtos en móvil:** beats de 3/4/6 celdas en la misma fila. Verificación de preview a 380px.

## No-objetivos (esta fase)

- Otras subdivisiones irregulares (quintillos, septillos como subdivisión de grid) — el modelo las admitiría añadiendo `kind`, pero no se implementan ahora.
- Cambiar la subdivisión global ×8/×16 sin resetear el grid: cambiar el selector global sigue reconstruyendo el grid (comportamiento actual). Documentar como reset conocido.
- Sticking L/R (las letras R/L de la foto): no afectan el sonido en una caja; no se modelan.
- Compases compuestos (6/8, 9/8, 12/8): el cálculo de beats = `num` los trataría como N beats de corchea, que no es la lectura agrupada habitual. Quedan fuera de foco; la referencia de esta fase son compases simples /4 (2/4, 3/4, 4/4).
