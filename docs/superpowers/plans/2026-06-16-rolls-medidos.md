# Rolls / Redobles Medidos — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Reemplazar el "roll" único aproximado del portal por una biblioteca completa de la familia de rolls (buzz, single, double, triple + medidos 5/6/7/9/10/11/13/15/17), con sonido por densidad correcta, acento de cierre, notación de trémolo y soporte en shorthand para transcripción de fotos.

**Architecture:** Single-file (`index.html`), sin build ni framework de tests. Se añade una tabla compartida `ROLL_TYPES` a nivel de script (la usan `DrumGridModule` y `ScoreLibraryModule`). El campo de celda `roll` reemplaza al antiguo `orn:'roll'`. El sonido se expande en N golpes derivados de la **duración de la celda** (nunca de una subdivisión fija) — esto hace que la Fase 2 (tresillos) funcione sin tocar el motor. Verificación vía el preview workflow del navegador.

**Tech Stack:** HTML/CSS/JS vanilla, abcjs v6.4.4 (render + síntesis). Verificación con las herramientas `preview_*`.

---

## Nota sobre verificación (sin framework de tests)

Este repo es un único `index.html` verificado con el **preview workflow** (igual que la Fase 1), no con unit tests. Cada tarea define una verificación concreta y automatable así:

1. `preview_start` (o reusar servidor) y cargar `index.html`.
2. `preview_eval` para fijar un estado determinista vía API pública (`DrumGridModule.setState(...)`, `ScoreLibraryModule.importJSON(...)`) y **leer el ABC generado** con `StateModule.get('abcCode')`.
3. Aserción sobre el string ABC (contar golpes, buscar decoraciones) y/o `preview_screenshot` para lo visual.

**Dato clave para las aserciones:** el snare (índice 7) se escribe como la nota ABC `c`. En un score **solo de snare**, el número de `c` en `abcCode` = número de golpes emitidos. A subdiv=8 la unidad es 1/32 y `durX = durH × 2`; un compás 4/4 completo = `durX` 32. Una celda `{ dur:8, durH:16 }` ocupa el compás entero (`durX=32`), suficiente para alojar hasta 32 golpes.

APIs públicas disponibles (ya exportadas): `DrumGridModule.getState()`, `DrumGridModule.setState(st)`, `StateModule.get('abcCode')`, `ScoreLibraryModule.importJSON(jsonStr)`.

---

## Task 1: Tabla `ROLL_TYPES`, estado `_curRoll`, migración de datos

**Files:**
- Modify: `index.html` (insertar `ROLL_TYPES` antes de `const DrumGridModule = (() => {`, línea ~1313; añadir `_curRoll` junto a `_curOrn` línea ~1344; añadir `_migrateCells` y llamarla en `setState` línea ~1761)

Esta tarea sienta el modelo de datos. Todavía no cambia sonido ni UI; solo introduce la tabla, el estado y la migración retro-compatible.

- [ ] **Step 1: Añadir la tabla compartida `ROLL_TYPES` a nivel de script**

Insertar EXACTAMENTE antes de la línea `const DrumGridModule = (() => {` (después del cierre `})();` del módulo anterior y su bloque de comentario):

```js
    /* ═══════════════════════════════════════
       ROLL TYPES — biblioteca de la familia de redobles
       Compartida por DrumGridModule (sonido/visual) y
       ScoreLibraryModule (parser de shorthand).
       strokes=null  → sostenido: llena la duración a densidad `hitSz`
       strokes=N     → medido: N golpes repartidos sobre la duración
       tag           → etiqueta corta (grid + botón); label → menú
       trem          → nº de rayas de trémolo (1/2/3) para la notación
    ═══════════════════════════════════════ */
    const ROLL_TYPES = {
      buzz:   { label: 'Buzz',      tag: 'Bz', group: 'Sostenidos', strokes: null, hitSz: 1, trem: 3 },
      single: { label: 'Single',    tag: 'Sg', group: 'Sostenidos', strokes: null, hitSz: 4, trem: 1 },
      double: { label: 'Double',    tag: 'Db', group: 'Sostenidos', strokes: null, hitSz: 2, trem: 3 },
      triple: { label: 'Triple',    tag: 'Tp', group: 'Sostenidos', strokes: null, hitSz: 2, trem: 3 },
      m5:  { label: '5 golpes',  tag: '5',  group: 'Medidos', strokes: 5,  trem: 2 },
      m6:  { label: '6 golpes',  tag: '6',  group: 'Medidos', strokes: 6,  trem: 2 },
      m7:  { label: '7 golpes',  tag: '7',  group: 'Medidos', strokes: 7,  trem: 3 },
      m9:  { label: '9 golpes',  tag: '9',  group: 'Medidos', strokes: 9,  trem: 2 },
      m10: { label: '10 golpes', tag: '10', group: 'Medidos', strokes: 10, trem: 2 },
      m11: { label: '11 golpes', tag: '11', group: 'Medidos', strokes: 11, trem: 2 },
      m13: { label: '13 golpes', tag: '13', group: 'Medidos', strokes: 13, trem: 3 },
      m15: { label: '15 golpes', tag: '15', group: 'Medidos', strokes: 15, trem: 3 },
      m17: { label: '17 golpes', tag: '17', group: 'Medidos', strokes: 17, trem: 3 }
    };
```

- [ ] **Step 2: Añadir el estado `_curRoll`**

En `DrumGridModule`, junto a `_curArt`/`_curOrn` (línea ~1344), añadir debajo de la línea `let _curOrn = null; ...`:

```js
      let _curRoll      = null;  // null | { type, accentRelease }  (selección activa del picker de roll)
```

- [ ] **Step 3: Añadir helper de migración y aplicarlo en `setState`**

Añadir esta función dentro de `DrumGridModule` (justo antes de `function _onGridChange()`, línea ~1643):

```js
      // Migra celdas antiguas: el formato viejo orn:'roll' (string) pasa al
      // nuevo campo roll:{type:'buzz'}. Idempotente. Muta y devuelve `cells`.
      function _migrateCells(cells) {
        if (!Array.isArray(cells)) return cells;
        for (const meas of cells) {
          if (!Array.isArray(meas)) continue;
          for (const col of meas) {
            if (!Array.isArray(col)) continue;
            for (const cell of col) {
              if (cell && typeof cell === 'object' && cell.orn === 'roll') {
                delete cell.orn;
                cell.roll = { type: 'buzz', accentRelease: true };
              }
            }
          }
        }
        return cells;
      }
```

Luego, en `setState` (línea ~1762), cambiar la hidratación de `_cells` para migrar:

```js
        setState(st) {
          _cells   = _migrateCells(JSON.parse(JSON.stringify(st.cells)));
          _measures = st.measures;
          _subdiv  = st.subdiv;
          _timeSig = { ...st.timeSig };
```

(El resto de `setState` queda igual.)

- [ ] **Step 4: Verificar la migración (preview)**

`preview_start` cargando `index.html`. Luego `preview_eval`:

```js
DrumGridModule.setState({
  measures: 1, subdiv: 8, timeSig: { num: 4, den: 4 },
  cells: (() => {
    const c = Array.from({length:1}, () => Array.from({length:8}, () => Array(10).fill(null)));
    c[0][0][7] = { dur: 1, durH: 2, orn: 'roll' };   // formato VIEJO
    return c;
  })()
});
const cell = DrumGridModule.getState().cells[0][0][7];
JSON.stringify(cell);
```

Esperado: `{"dur":1,"durH":2,"roll":{"type":"buzz","accentRelease":true}}` — sin `orn`, con `roll`.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat(rolls): ROLL_TYPES table, _curRoll state, legacy orn:'roll' migration"
```

---

## Task 2: Expansión de sonido por tipo de roll

**Files:**
- Modify: `index.html` (añadir `_expandRoll` cerca de `fmtLen`/`emitRest` ~1445; ampliar recogida de eventos ~1467 y ~1480-1484; reescribir la rama de emisión del roll ~1510-1522)

Esta tarea hace que cada tipo de roll suene con su densidad correcta y con acento de cierre en los medidos. El espaciado se deriva de `durX`.

- [ ] **Step 1: Añadir la función `_expandRoll`**

Colócala como función a nivel de `DrumGridModule` (no necesita estado del módulo), junto a `_migrateCells`, antes de `function _onGridChange()` (línea ~1643). Al ser de ámbito de módulo es visible desde `buildVoice`, que está anidada en `_buildABC`:

```js
      // Expande un roll en una lista de duraciones de golpe (en unidades de 1/32)
      // que suman exactamente durX. Medidos: N golpes (último absorbe el resto).
      // Sostenidos: golpes de tamaño hitSz llenando durX (+ golpe corto sobrante).
      // El espaciado SIEMPRE deriva de durX — nunca de una subdivisión fija
      // (esto permite tresillos en Fase 2 sin cambiar el motor).
      function _expandRoll(durX, type) {
        const def = ROLL_TYPES[type] || ROLL_TYPES.buzz;
        let hitSz, hits;
        if (def.strokes != null) {
          hits  = Math.min(def.strokes, durX);
          hitSz = Math.max(1, Math.floor(durX / def.strokes));
        } else {
          hitSz = Math.max(1, Math.min(def.hitSz, durX));
          hits  = Math.floor(durX / hitSz);
        }
        const sizes = [];
        let used = 0;
        for (let h = 0; h < hits; h++) { sizes.push(hitSz); used += hitSz; }
        const rem = durX - used;
        if (rem > 0) {
          if (def.strokes != null) sizes[sizes.length - 1] += rem; // medido: el cierre se alarga
          else sizes.push(rem);                                    // sostenido: golpe corto extra
        }
        return sizes;
      }
```

- [ ] **Step 2: Recoger el campo `roll` en los eventos**

En `buildVoice`, en el `notesHere.push(...)` (línea ~1467), añadir `roll: cell.roll`:

```js
                    notesHere.push({ note: INSTRUMENTS[i].note, dur: cell.dur, durX, art: cell.art, orn: cell.orn, roll: cell.roll });
```

Y donde se calcula el evento fusionado (líneas ~1480-1484), añadir la detección de roll:

```js
                const art   = notesHere.find(n => n.art)?.art || null;
                const ornEv = notesHere.find(n => n.orn);
                const orn   = ornEv?.orn || null;
                const rollEv = notesHere.find(n => n.roll);
                const roll   = rollEv?.roll || null;
                const graceNote = ornEv ? ornEv.note : notesHere[0].note;
                events.push({ absX, notes: notesHere.map(n => n.note), dur, durX, art, orn, roll, graceNote });
```

- [ ] **Step 3: Reescribir la rama de emisión del roll**

Reemplazar el bloque actual (líneas ~1510-1522, desde `if (ev.orn === 'roll') {` hasta el `}` que cierra el `else`) por:

```js
                if (ev.roll) {
                  const sizes = _expandRoll(ev.durX, ev.roll.type);
                  const def   = ROLL_TYPES[ev.roll.type] || ROLL_TYPES.buzz;
                  const last  = sizes.length - 1;
                  const accRelease = ev.roll.accentRelease && def.strokes != null;
                  for (let h = 0; h < sizes.length; h++) {
                    const isFirst = h === 0;
                    const pre    = isFirst ? prefix : '';
                    const dOpen  = isFirst ? deco : '';
                    const dClose = isFirst ? decoClose : '';
                    const trem   = isFirst ? `!trem${def.trem}!` : '';
                    const acc    = (h === last && accRelease) ? '!accent!' : '';
                    voice += pre + dOpen + acc + trem + chord + fmtLen(sizes[h]) + dClose;
                  }
                  posX += ev.durX;
                } else {
                  voice += prefix + deco + chord + fmtLen(ev.durX) + decoClose;
                }
```

- [ ] **Step 4: Verificar el conteo de golpes y el acento de cierre (preview)**

Recargar la página (`preview_eval: window.location.reload()`), luego `preview_eval`:

```js
function rollAbc(type) {
  DrumGridModule.setState({
    measures: 1, subdiv: 8, timeSig: { num: 4, den: 4 },
    cells: (() => {
      const c = Array.from({length:1}, () => Array.from({length:8}, () => Array(10).fill(null)));
      c[0][0][7] = { dur: 8, durH: 16, roll: { type, accentRelease: true } }; // durX=32, todo el compás
      return c;
    })()
  });
  return StateModule.get('abcCode');
}
const counts = {};
for (const t of ['m5','m7','m9','m17','single','double','buzz']) {
  counts[t] = (rollAbc(t).match(/c/g) || []).length;
}
const m5HasAccent = /!accent!c\d*\|/.test(rollAbc('m5'));
JSON.stringify({ counts, m5HasAccent });
```

Esperado: `counts.m5===5`, `counts.m7===7`, `counts.m9===9`, `counts.m17===17`, `counts.single===8`, `counts.double===16`, `counts.buzz===32`, y `m5HasAccent===true` (el último golpe lleva `!accent!`).

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat(rolls): per-type hit expansion with release accent (sound)"
```

---

## Task 3: Notación visual — etiqueta de roll en el grid + validación de trémolo

**Files:**
- Modify: `index.html` (bloque de `_renderTable` que pinta `orn-tag`, líneas ~1580-1587)

El `!trem{N}!` ya se emite (Task 2). Esta tarea pinta la etiqueta del tipo de roll en la celda del grid y valida cómo se ve el trémolo en la partitura, con fallback si abcjs no lo dibuja bien.

- [ ] **Step 1: Pintar la etiqueta del roll en la celda del grid**

En `_renderTable`, reemplazar el bloque (líneas ~1580-1587):

```js
              if (isNote) {
                if (raw.art === 'accent')     cls += ' art-accent';
                else if (raw.art === 'ghost') cls += ' art-ghost';
                if (raw.orn) {
                  const tag = ({ flam: 'F', drag: 'D', ruff: 'Rf', roll: 'Ro' })[raw.orn] || '';
                  inner = `<span class="orn-tag">${tag}</span>`;
                }
              }
```

por:

```js
              if (isNote) {
                if (raw.art === 'accent')     cls += ' art-accent';
                else if (raw.art === 'ghost') cls += ' art-ghost';
                let tag = '';
                if (raw.roll)     tag = (ROLL_TYPES[raw.roll.type] || {}).tag || 'R';
                else if (raw.orn) tag = ({ flam: 'F', drag: 'D', ruff: 'Rf' })[raw.orn] || '';
                if (tag) inner = `<span class="orn-tag">${tag}</span>`;
              }
```

- [ ] **Step 2: Verificar la etiqueta en el grid (preview)**

`preview_eval` para colocar un m7 en snare y leer el HTML de la celda:

```js
DrumGridModule.setState({
  measures: 1, subdiv: 8, timeSig: { num: 4, den: 4 },
  cells: (() => {
    const c = Array.from({length:1}, () => Array.from({length:8}, () => Array(10).fill(null)));
    c[0][0][7] = { dur: 2, durH: 4, roll: { type: 'm7', accentRelease: true } };
    return c;
  })()
});
document.querySelector('.dg-cell[data-m="0"][data-c="0"][data-i="7"]').innerHTML;
```

Esperado: contiene `<span class="orn-tag">7</span>`.

- [ ] **Step 3: Validar el render de trémolo en la partitura y decidir fallback**

`preview_screenshot` de la partitura con el m7 colocado (del paso anterior). Inspeccionar visualmente:

- **Si** la nota muestra rayas de trémolo / se ve como un redoble → dejar el `!trem{N}!` tal cual. La tarea termina aquí.
- **Si** abcjs ignora `!trem{N}!` o lanza warning de decoración → aplicar fallback: en la rama de emisión del roll (Task 2, Step 3), cambiar la línea del trémolo por el marcador heredado:

```js
                    const trem   = isFirst ? '!///!' : '';
```

Volver a `preview_screenshot` y confirmar que al menos hay un marcador visual sobre el primer golpe. Documentar en el commit qué variante quedó.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat(rolls): grid roll tag + validated tremolo notation"
```

---

## Task 4: UX — selector de rolls (desplegable)

**Files:**
- Modify: `index.html` (CSS del popover tras `.orn-btn` ~línea 380; HTML: reemplazar el botón `#btn-orn-roll` ~línea 907; JS: `ORN_BTNS` y `_setOrn` ~1710-1719, handler de click de celda ~1606-1629, y nuevo bloque de wiring del picker dentro de `init()`)

Reemplaza el botón único "Roll" por un desplegable con secciones Sostenidos/Medidos y toggle de acento. Selección mutuamente excluyente con los ornamentos de gracia.

- [ ] **Step 1: CSS del desplegable**

Tras la regla `.orn-btn svg { display: block; }` (línea ~380), añadir:

```css
    /* Roll picker (desplegable de tipos de redoble) */
    .roll-picker { position: relative; display: inline-block; }
    .roll-label { font-size: 0.7rem; margin-left: 3px; vertical-align: middle; }
    .roll-menu {
      position: absolute; top: 100%; right: 0; z-index: 50;
      margin-top: 4px; padding: 8px;
      background: var(--bg-base); border: 1px solid var(--border);
      border-radius: 6px; box-shadow: 0 4px 16px rgba(0,0,0,0.45);
      min-width: 190px;
    }
    .roll-menu[hidden] { display: none; }
    .roll-menu-sec {
      font-size: 0.62rem; text-transform: uppercase; letter-spacing: 0.04em;
      color: var(--text-muted); margin: 6px 0 3px;
    }
    .roll-menu-sec:first-child { margin-top: 0; }
    .roll-menu-row { display: flex; flex-wrap: wrap; gap: 4px; }
    .roll-opt {
      background: var(--bg-base); color: var(--text-muted);
      border: 1px solid var(--border); border-radius: 4px;
      padding: 3px 8px; font-size: 0.78rem; cursor: pointer; min-width: 32px;
      font-family: inherit; transition: color 0.12s, border-color 0.12s, background 0.12s;
    }
    .roll-opt:hover  { border-color: var(--accent); color: var(--accent); }
    .roll-opt.active { background: var(--accent); color: #0d1117; border-color: var(--accent); font-weight: 600; }
    .roll-accent {
      display: flex; align-items: center; gap: 5px; margin-top: 9px;
      font-size: 0.72rem; color: var(--text-muted); cursor: pointer;
    }
```

- [ ] **Step 2: Reemplazar el botón de roll en el HTML**

Sustituir la línea del botón `#btn-orn-roll` (línea ~907) por el shell del picker (el `<div id="roll-menu">` se rellena por JS desde `ROLL_TYPES`):

```html
            <span class="roll-picker">
              <button id="btn-roll" class="dur-btn orn-btn" title="Roll / redoble — elegir tipo" aria-haspopup="true" aria-expanded="false"><svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 26 22" width="26" height="22"><ellipse cx="10" cy="17" rx="5" ry="3.3" transform="rotate(-18,10,17)" fill="currentColor"/><line x1="14.7" y1="15.5" x2="14.7" y2="2" stroke="currentColor" stroke-width="1.5"/><line x1="11" y1="12" x2="18" y2="9.5" stroke="currentColor" stroke-width="1.3"/><line x1="11" y1="9.5" x2="18" y2="7" stroke="currentColor" stroke-width="1.3"/><line x1="11" y1="7" x2="18" y2="4.5" stroke="currentColor" stroke-width="1.3"/></svg><span id="roll-label" class="roll-label"></span></button>
              <div id="roll-menu" class="roll-menu" hidden></div>
            </span>
```

- [ ] **Step 3: Quitar `roll` de `ORN_BTNS` y hacer que `_setOrn` limpie el roll**

Reemplazar el bloque del selector de ornamento (líneas ~1709-1719):

```js
          /* Ornament selector — single choice */
          const ORN_BTNS = { flam: 'btn-orn-flam', drag: 'btn-orn-drag', ruff: 'btn-orn-ruff', roll: 'btn-orn-roll' };
          function _setOrn(val) {
            _curOrn = (_curOrn === val) ? null : val;
            for (const [k, id] of Object.entries(ORN_BTNS)) {
              document.getElementById(id).classList.toggle('active', _curOrn === k);
            }
          }
          Object.entries(ORN_BTNS).forEach(([k, id]) => {
            document.getElementById(id).addEventListener('click', () => _setOrn(k));
          });
```

por:

```js
          /* Ornament selector (grace notes) — single choice */
          const ORN_BTNS = { flam: 'btn-orn-flam', drag: 'btn-orn-drag', ruff: 'btn-orn-ruff' };
          function _setOrn(val) {
            _curOrn = (_curOrn === val) ? null : val;
            for (const [k, id] of Object.entries(ORN_BTNS)) {
              document.getElementById(id).classList.toggle('active', _curOrn === k);
            }
            if (_curOrn) { _curRoll = null; _renderRollState(); }  // exclusión mutua con roll
          }
          Object.entries(ORN_BTNS).forEach(([k, id]) => {
            document.getElementById(id).addEventListener('click', () => _setOrn(k));
          });

          /* Roll picker — construye el menú desde ROLL_TYPES y gestiona la selección */
          const rollBtn   = document.getElementById('btn-roll');
          const rollMenu  = document.getElementById('roll-menu');
          const rollLabel = document.getElementById('roll-label');
          (function _buildRollMenu() {
            const groups = {};
            for (const [type, def] of Object.entries(ROLL_TYPES)) {
              (groups[def.group] ||= []).push([type, def]);
            }
            let html = '';
            for (const [g, items] of Object.entries(groups)) {
              html += `<div class="roll-menu-sec">${g}</div><div class="roll-menu-row">`;
              for (const [type, def] of items) {
                html += `<button type="button" class="roll-opt" data-roll="${type}" title="${def.label}">${def.tag}</button>`;
              }
              html += `</div>`;
            }
            html += `<label class="roll-accent"><input type="checkbox" id="roll-accent-cb" checked> Acento en cierre</label>`;
            rollMenu.innerHTML = html;
          })();
          const rollAccentCb = document.getElementById('roll-accent-cb');
          function _renderRollState() {
            rollBtn.classList.toggle('active', !!_curRoll);
            rollLabel.textContent = _curRoll ? ((ROLL_TYPES[_curRoll.type] || {}).tag || '') : '';
            rollMenu.querySelectorAll('.roll-opt').forEach(b =>
              b.classList.toggle('active', !!_curRoll && b.dataset.roll === _curRoll.type));
          }
          function _setRoll(type) {
            if (_curRoll && _curRoll.type === type) {
              _curRoll = null;
            } else {
              _curRoll = { type, accentRelease: rollAccentCb.checked };
              _curOrn = null;  // exclusión mutua con ornamentos de gracia
              for (const id of Object.values(ORN_BTNS)) document.getElementById(id).classList.remove('active');
            }
            _renderRollState();
          }
          rollBtn.addEventListener('click', (e) => {
            e.stopPropagation();
            rollMenu.hidden = !rollMenu.hidden;
            rollBtn.setAttribute('aria-expanded', String(!rollMenu.hidden));
          });
          rollMenu.querySelectorAll('.roll-opt').forEach(b =>
            b.addEventListener('click', (e) => { e.stopPropagation(); _setRoll(b.dataset.roll); }));
          rollAccentCb.addEventListener('change', () => {
            if (_curRoll) _curRoll.accentRelease = rollAccentCb.checked;
          });
          document.addEventListener('click', (e) => {
            if (!rollMenu.hidden && !rollMenu.contains(e.target) && e.target !== rollBtn && !rollBtn.contains(e.target)) {
              rollMenu.hidden = true;
              rollBtn.setAttribute('aria-expanded', 'false');
            }
          });
```

Nota: `_renderRollState` se referencia desde `_setOrn`; al estar ambas declaradas con `function`/closure dentro del mismo `init()`, el hoisting las hace visibles. Mantener `_setOrn` y el bloque del roll picker dentro de la misma función `init()`.

- [ ] **Step 4: Soportar `_curRoll` al colocar y editar notas**

En el handler de click de celda, reemplazar la rama de celda existente (líneas ~1606-1614):

```js
            if (raw && typeof raw === 'object') {
              if (_curArt || _curOrn) {
                // A modifier toggle is active → apply/toggle it on the existing note
                if (_curArt) raw.art = (raw.art === _curArt) ? undefined : _curArt;
                if (_curOrn) raw.orn = (raw.orn === _curOrn) ? undefined : _curOrn;
              } else {
                // Toggle off an existing note
                _cells[m][c][i] = null;
              }
            } else {
```

por:

```js
            if (raw && typeof raw === 'object') {
              if (_curArt || _curOrn || _curRoll) {
                // A modifier toggle is active → apply/toggle it on the existing note
                if (_curArt)  raw.art  = (raw.art === _curArt) ? undefined : _curArt;
                if (_curOrn)  raw.orn  = (raw.orn === _curOrn) ? undefined : _curOrn;
                if (_curRoll) raw.roll = (raw.roll && raw.roll.type === _curRoll.type) ? undefined : { ..._curRoll };
              } else {
                // Toggle off an existing note
                _cells[m][c][i] = null;
              }
            } else {
```

Y en la rama de nota nueva (líneas ~1625-1628):

```js
                const note = { dur, durH };
                if (_curArt) note.art = _curArt;
                if (_curOrn) note.orn = _curOrn;
                _cells[m][c][i] = note;
```

por:

```js
                const note = { dur, durH };
                if (_curArt)  note.art  = _curArt;
                if (_curOrn)  note.orn  = _curOrn;
                if (_curRoll) note.roll = { ..._curRoll };
                _cells[m][c][i] = note;
```

- [ ] **Step 5: Verificar el picker (preview)**

`preview_eval: window.location.reload()`. Luego, simular la selección y colocación con la API real del DOM:

```js
document.getElementById('btn-roll').click();                 // abre el menú
document.querySelector('.roll-opt[data-roll="m7"]').click();  // elige 7 golpes
const labelOk = document.getElementById('roll-label').textContent === '7';
const menuClosedAfterPick = !document.getElementById('roll-menu').hidden; // sigue abierto tras elegir (ok)
// colocar en snare, celda 0
const cell = document.querySelector('.dg-cell[data-m="0"][data-c="0"][data-i="7"]');
cell.click();
const placed = DrumGridModule.getState().cells[0][0][7];
JSON.stringify({ labelOk, placedType: placed && placed.roll && placed.roll.type });
```

Esperado: `labelOk===true` y `placedType==="m7"`. Además `preview_screenshot` para confirmar que el desplegable se ve bien y no desborda.

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "feat(rolls): dropdown roll-type picker with mutual exclusion + placement"
```

---

## Task 5: Shorthand — `r`→buzz, `mods.roll`, validación

**Files:**
- Modify: `index.html` (`_parseShorthand` en `ScoreLibraryModule`: `CHAR_MOD` ~1954, clonado de roll en el loop de tracks ~1977, loop de `mods` ~1990-1991; comentario de doc ~1950-1951)

Hace que el formato que yo emito al transcribir fotos soporte tipos de roll específicos vía `mods`, y migra el `r`/`orn:'roll'` antiguos.

- [ ] **Step 1: Actualizar `CHAR_MOD` y el comentario de doc**

Reemplazar el comentario (líneas ~1950-1951) y el `CHAR_MOD` (líneas ~1954-1957):

```js
      // Per-position chars: '.'=silence  'x'=note  'X'=accent  'g'=ghost
      //                     'f'=flam  'd'=drag  'r'=roll(buzz)   (ruff y rolls específicos via mods)
      function _parseShorthand(data) {
        const ABBR = { CR:0, RD:1, RB:2, HH:3, HO:4, T1:5, T2:6, SN:7, TF:8, KI:9 };
        const CHAR_MOD = {
          'x': {}, 'X': { art: 'accent' }, 'g': { art: 'ghost' },
          'f': { orn: 'flam' }, 'd': { orn: 'drag' },
          'r': { roll: { type: 'buzz', accentRelease: true } }
        };
```

- [ ] **Step 2: Clonar `roll` al crear la celda en el loop de tracks**

Reemplazar la línea de creación de celda (línea ~1977):

```js
            cells[Math.floor(pos / cols)][pos % cols][idx] = { dur: 1, durH: 2, ...mod };
```

por (evita compartir la referencia del objeto roll entre celdas):

```js
            const newCell = { dur: 1, durH: 2, ...mod };
            if (newCell.roll) newCell.roll = { ...newCell.roll };
            cells[Math.floor(pos / cols)][pos % cols][idx] = newCell;
```

- [ ] **Step 3: Soportar `roll` (y migrar `orn:'roll'`) en el loop de `mods`**

Reemplazar el bloque de aplicación de overrides (líneas ~1990-1991):

```js
            if (ov.art) cell.art = ov.art;
            if (ov.orn) cell.orn = ov.orn;
```

por:

```js
            if (ov.art) cell.art = ov.art;
            if (ov.orn === 'roll') {
              cell.roll = { type: 'buzz', accentRelease: true };   // migración formato viejo
              delete cell.orn;
            } else if (ov.orn) {
              cell.orn = ov.orn;
            }
            if (ov.roll) {
              if (!ROLL_TYPES[ov.roll.type]) {
                throw new Error(`Tipo de roll desconocido '${ov.roll.type}' en ${abbr} col ${pos}`);
              }
              cell.roll = { type: ov.roll.type, accentRelease: ov.roll.accentRelease !== false };
              delete cell.orn;
            }
```

(`ROLL_TYPES` es visible aquí por estar a nivel de script — Task 1, Step 1.)

- [ ] **Step 4: Verificar import de shorthand con rolls (preview)**

`preview_eval: window.location.reload()`. Luego:

```js
const json = JSON.stringify({
  name: "Test rolls", format: "shorthand", timeSig: "4/4", subdiv: 8, measures: 1,
  tracks: { SN: "x...r...", KI: "x...x..." },
  mods: { SN: { "0": { roll: { type: "m5" } }, "4": { roll: { type: "m7", accentRelease: false } } } }
});
ScoreLibraryModule.importJSON(json);
const cells = DrumGridModule.getState().cells;
const r0 = cells[0][0][7].roll, r4 = cells[0][4][7].roll;
JSON.stringify({ r0, r4 });
```

Esperado: `r0` = `{"type":"m5","accentRelease":true}` (el `r` de la pista fue sobrescrito por mods), `r4` = `{"type":"m7","accentRelease":false}`.

Luego verificar el error con tipo inválido:

```js
let errMsg = null;
try {
  ScoreLibraryModule.importJSON(JSON.stringify({
    name: "Bad", format: "shorthand", timeSig: "4/4", subdiv: 8, measures: 1,
    tracks: { SN: "x......." },
    mods: { SN: { "0": { roll: { type: "m99" } } } }
  }));
} catch (e) { errMsg = e.message; }
errMsg;
```

Nota: `_importJSON` captura el error y muestra `alert`; el valor de retorno es `false`. Para confirmar el mensaje, comprobar en su lugar que el import devolvió `false`:

```js
ScoreLibraryModule.importJSON(JSON.stringify({
  name: "Bad", format: "shorthand", timeSig: "4/4", subdiv: 8, measures: 1,
  tracks: { SN: "x......." },
  mods: { SN: { "0": { roll: { type: "m99" } } } }
}));
```

Esperado: devuelve `false` (tipo `m99` rechazado).

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat(rolls): shorthand r=buzz + mods.roll with type validation"
```

---

## Task 6: SKILL.md — tabla de tipos de roll + transcripción desde foto

**Files:**
- Modify: `C:\Users\JosuReborn\AppData\Roaming\Claude\local-agent-mode-sessions\skills-plugin\b11912fc-1893-424f-8615-dd0734550cf4\2a2d206d-faf2-49e4-8b24-7ea3b5965f5a\skills\tutor-de-bateria-avanzado\SKILL.md` (sección de shorthand, tras la tabla de `mods` ~línea 76)

El tutor debe saber emitir los tipos de roll correctos al transcribir una foto.

- [ ] **Step 1: Añadir la subsección de rolls tras el bloque de `mods`**

Tras el bloque de ejemplo de `mods` (línea ~76, después del bloque ```json ... ``` de ruff) e inmediatamente antes de `**Reglas:**`, insertar:

```markdown
**Redobles (rolls):** el char `r` en `tracks` es un roll genérico (buzz). Para un redoble específico, usa `mods[...].roll`:

\```json
"mods": { "SN": { "0": { "roll": { "type": "m5" } }, "4": { "roll": { "type": "m7", "accentRelease": true } } } }
\```

| `type`   | Redoble |
|----------|---------|
| `buzz`   | Buzz / press roll (rebotes densos) |
| `single` | Single-stroke roll |
| `double` | Double-stroke / redoble largo |
| `triple` | Triple-stroke roll |
| `m5` `m6` `m7` `m9` `m10` `m11` `m13` `m15` `m17` | Redobles medidos de 5…17 golpes |

`accentRelease` (default `true`) acentúa el golpe de cierre de los medidos; ponlo en `false` para quitarlo.

**Al transcribir una foto con un redoble:** identifica el tipo contando las rayas de trémolo (1=corcheas, 2=semicorcheas, 3=fusas) y el número/nombre del rudimento (p. ej. "5", "7", "9" golpes). Emite el `type` correspondiente. Si es un trémolo sostenido sin número (redoble largo), usa `double`; si es un crush de rebotes, `buzz`.
```

(Las secuencias `\``` representan los triples backticks literales del bloque JSON dentro del Markdown.)

- [ ] **Step 2: Verificar el contenido**

Leer el archivo y confirmar que la tabla de tipos de roll y la guía de transcripción están presentes tras el bloque de `mods` y antes de `**Reglas:**`.

- [ ] **Step 3: Commit**

```bash
git add "C:/Users/JosuReborn/AppData/Roaming/Claude/local-agent-mode-sessions/skills-plugin/b11912fc-1893-424f-8615-dd0734550cf4/2a2d206d-faf2-49e4-8b24-7ea3b5965f5a/skills/tutor-de-bateria-avanzado/SKILL.md"
git commit -m "docs(tutor): roll type table + photo transcription guidance"
```

---

## Verificación final (tras todas las tareas)

1. **Retrocompat:** cargar un score viejo (con `orn:'roll'` o sin rolls) desde la biblioteca → no rompe; el roll viejo se ve como buzz.
2. **End-to-end manual:** abrir el portal en el navegador, elegir varios tipos de roll del desplegable, colocarlos, reproducir con ▶ y confirmar audiblemente que un `m5` suena menos denso que un `m17` y que el buzz es un crush.
3. **Persistencia:** guardar en biblioteca, recargar la página, reabrir → los tipos de roll se conservan.
4. **Responsive:** `preview_resize` a ~380px y confirmar que el desplegable no desborda.

Luego usar **superpowers:finishing-a-development-branch** para cerrar el trabajo.
```
