# Articulaciones y Ornamentos de Batería — Implementation Plan (Fase 1)

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Añadir acento, ghost note, flam, drag, ruff y roll a las notas del portal de batería — en el grid, la partitura abcjs, y el formato JSON/shorthand.

**Architecture:** Todo vive en el archivo único `index.html` dentro de `DrumGridModule` (IIFE). Se extiende el objeto de celda con campos opcionales `art` y `orn`; el generador ABC (`_buildABC`/`buildVoice`) emite las decoraciones/grace-notes correspondientes; el panel gana botones toggle; el parser shorthand gana vocabulario nuevo. Retrocompatible: celdas sin los campos nuevos se comportan idéntico a hoy.

**Tech Stack:** HTML/CSS/JS vanilla, abcjs v6.4.4 (ABC → SVG + síntesis MIDI), single-file. Sin test runner: la verificación es por el workflow de preview (`preview_start`, `preview_eval`, `preview_screenshot`) leyendo el ABC generado vía `StateModule.get('abcCode')` y observando el render.

**Verificación — helper reutilizable.** En las tareas que verifican ABC, primero define este helper en la consola del preview con `preview_eval`:

```js
window.__mk = function (art, orn, inst) {
  // inst index: SN=7, KI=9, HH=3. Default snare (pitch 'c').
  const i = inst ?? 7;
  const cell = { dur: 1, durH: 2 };
  if (art) cell.art = art;
  if (orn) cell.orn = orn;
  const cells = [Array.from({ length: 8 }, () => Array(10).fill(null))];
  cells[0][0][i] = cell;
  DrumGridModule.setState({ measures: 1, subdiv: 8, timeSig: { num: 4, den: 4 }, cells });
  return StateModule.get('abcCode');
};
```

Snare (índice 7) tiene pitch ABC `c`, así que las aserciones buscan tokens junto a `c`.

---

## Task 1: Motor ABC — campos de celda, decoraciones de header, emisión

**Files:**
- Modify: `index.html` (dentro de `DrumGridModule`: declaración de estado ~1319-1322; `_buildABC` header ~1392-1404; recolección de eventos ~1436-1456; emisión ~1470-1474)

- [ ] **Step 1: Añadir variables de estado de modificadores**

En `index.html`, localiza (≈1319-1322):

```js
      let _curDurFrac16 = 2;  // selected duration in 16ths (1=16th, 2=8th, 4=quarter, 8=half, 16=whole)
      let _isDotted     = false;
      // _cells[measure][col][instIdx] = null | 'rest' | { dur: N }
      let _cells    = [];
```

Reemplázalo por:

```js
      let _curDurFrac16 = 2;  // selected duration in 16ths (1=16th, 2=8th, 4=quarter, 8=half, 16=whole)
      let _isDotted     = false;
      let _curArt       = null;  // null | 'accent' | 'ghost'  (latching toggle)
      let _curOrn       = null;  // null | 'flam' | 'drag' | 'ruff' | 'roll'  (single-choice)
      // _cells[measure][col][instIdx] = null | 'rest' | { dur, durH, art?, orn? }
      let _cells    = [];
```

- [ ] **Step 2: Añadir la boilerplate de ghost notes al header ABC**

Localiza el array `header` en `_buildABC` (≈1397-1404):

```js
        const header = [
          'X:1', 'T:Groove',
          `M:${_timeSig.num}/${_timeSig.den}`, `L:${L}`,
          `Q:${StateModule.get('tempoBpm') || 120}`,
          percmap.trim(),
          '%%score (V:1 V:2)',
          'K:C clef=perc', ''
        ].join('\n');
```

Reemplázalo por (añade las dos líneas `%%deco` de paréntesis ghost, tomadas literalmente de la guía de Lou Montulli):

```js
        const header = [
          'X:1', 'T:Groove',
          `M:${_timeSig.num}/${_timeSig.den}`, `L:${L}`,
          `Q:${StateModule.get('tempoBpm') || 120}`,
          percmap.trim(),
          '%%deco (. 0 a 5 1 1 "@-8,-3("',
          '%%deco ). 0 a 5 1 1 "@4,-3)"',
          '%%score (V:1 V:2)',
          'K:C clef=perc', ''
        ].join('\n');
```

- [ ] **Step 3: Transportar `art`/`orn` al recolectar eventos**

Localiza el bloque de recolección (≈1436-1456):

```js
              for (const i of idxs) {
                if (nextFreeAt[i] <= absC) {
                  const cell = _cells[m]?.[c]?.[i];
                  if (cell && typeof cell === 'object') {
                    // Legacy cells without durH use dur * cellIn32 as durX
                    const durH = cell.durH ?? cell.dur * 2;
                    const durX = durH * halfLIn32;
                    notesHere.push({ note: INSTRUMENTS[i].note, dur: cell.dur, durX });
                  }
                }
              }
              if (notesHere.length > 0) {
                const dur  = Math.max(...notesHere.map(n => n.dur));
                const durX = Math.max(...notesHere.map(n => n.durX));
                for (const i of idxs) {
                  if (nextFreeAt[i] <= absC && _cells[m]?.[c]?.[i] && typeof _cells[m][c][i] === 'object') {
                    nextFreeAt[i] = absC + dur;
                  }
                }
                events.push({ absX, notes: notesHere.map(n => n.note), dur, durX });
              }
```

Reemplázalo por:

```js
              for (const i of idxs) {
                if (nextFreeAt[i] <= absC) {
                  const cell = _cells[m]?.[c]?.[i];
                  if (cell && typeof cell === 'object') {
                    // Legacy cells without durH use dur * cellIn32 as durX
                    const durH = cell.durH ?? cell.dur * 2;
                    const durX = durH * halfLIn32;
                    notesHere.push({ note: INSTRUMENTS[i].note, dur: cell.dur, durX, art: cell.art, orn: cell.orn });
                  }
                }
              }
              if (notesHere.length > 0) {
                const dur  = Math.max(...notesHere.map(n => n.dur));
                const durX = Math.max(...notesHere.map(n => n.durX));
                for (const i of idxs) {
                  if (nextFreeAt[i] <= absC && _cells[m]?.[c]?.[i] && typeof _cells[m][c][i] === 'object') {
                    nextFreeAt[i] = absC + dur;
                  }
                }
                // First non-null modifier wins for the merged chord event.
                const art   = notesHere.find(n => n.art)?.art || null;
                const ornEv = notesHere.find(n => n.orn);
                const orn   = ornEv?.orn || null;
                const graceNote = ornEv?.note || notesHere[0].note;
                events.push({ absX, notes: notesHere.map(n => n.note), dur, durX, art, orn, graceNote });
              }
```

- [ ] **Step 4: Emitir grace notes y decoraciones**

Localiza la emisión del acorde (≈1470-1474):

```js
              if (ev && ev.absX === posX) {
                evIdx++;
                const chord = ev.notes.length === 1 ? ev.notes[0] : '[' + ev.notes.join('') + ']';
                voice += chord + fmtLen(ev.durX);
                posX  += ev.durX;
              } else if (ev && ev.absX > posX && ev.absX < measEndX) {
```

Reemplázalo por:

```js
              if (ev && ev.absX === posX) {
                evIdx++;
                let prefix = '';
                const g = ev.graceNote;
                if (ev.orn === 'flam')      prefix += `{/${g}}`;
                else if (ev.orn === 'drag') prefix += `{${g}${g}}`;
                else if (ev.orn === 'ruff') prefix += `{${g}${g}${g}}`;
                let deco = '';
                if (ev.art === 'accent')     deco += '!accent!';
                else if (ev.art === 'ghost') deco += '!(.!!).!';
                if (ev.orn === 'roll')       deco += '!///!';
                const chord = ev.notes.length === 1 ? ev.notes[0] : '[' + ev.notes.join('') + ']';
                voice += prefix + deco + chord + fmtLen(ev.durX);
                posX  += ev.durX;
              } else if (ev && ev.absX > posX && ev.absX < measEndX) {
```

- [ ] **Step 5: Verificar la generación de ABC**

Inicia el preview (`preview_start`) y carga el portal. En `preview_eval` define `window.__mk` (ver helper arriba), luego ejecuta cada llamada y comprueba el substring del ABC devuelto:

```js
__mk('accent', null);  // debe contener  !accent!c
__mk('ghost',  null);  // debe contener  !(.!!).!c
__mk(null, 'flam');    // debe contener  {/c}c
__mk(null, 'drag');    // debe contener  {cc}c
__mk(null, 'ruff');    // debe contener  {ccc}c
__mk(null, 'roll');    // debe contener  !///!c
__mk('accent', 'flam');// debe contener  {/c}!accent!c
```

Expected: cada string contiene el token indicado.

- [ ] **Step 6: Verificar el render del score (riesgo: ghost + roll)**

Tras `__mk('ghost', null)`, toma `preview_screenshot` del área del score y confirma que la nota de snare aparece **entre paréntesis**. Revisa `preview_console_logs` por errores de abcjs.

- Si hay error/no aparecen paréntesis para ghost: fallback — en Step 4 cambia `deco += '!(.!!).!'` por `deco += '!(!'` antes y `prefix`/sufijo de anotación `"@( )"`; si tampoco, deja ghost como **visual-only en el grid** (Task 4) y emite el acorde sin decoración de paréntesis. Documenta la decisión.
- Tras `__mk(null, 'roll')`, si la consola muestra "unknown decoration `///`": fallback — elimina la línea `if (ev.orn === 'roll') deco += '!///!';` (el roll quedará marcado solo en el grid vía su badge en Task 4). Documenta la decisión.

Confirma que `__mk('accent', null)` renderiza el símbolo de acento sin errores.

- [ ] **Step 7: Commit**

```bash
git add index.html
git commit -m "feat: ABC emission for drum articulations and ornaments

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```

---

## Task 2: Colocación — adjuntar modificadores a notas nuevas y a notas existentes

**Files:**
- Modify: `index.html` (handler de click de celda ~1544-1565)

- [ ] **Step 1: Adjuntar modificadores al colocar nota nueva y aplicar a notas existentes**

Localiza el handler de click (≈1544-1565):

```js
          cell.addEventListener('click', (e) => {
            e.preventDefault();
            const m = +cell.dataset.m, c = +cell.dataset.c, i = +cell.dataset.i;
            const raw = _cells[m][c][i];
            if (raw && typeof raw === 'object') {
              // Toggle off an existing note
              _cells[m][c][i] = null;
            } else {
              // If a previous long note covers this cell, clear that source note instead
              const occupant = _getOccupantCell(m, c, i);
              if (occupant) {
                _cells[occupant.m][occupant.c][i] = null;
              } else if (raw === null) {
                // Place new note; clamp to measure boundary
                const maxDurH = (_colsPerMeasure() - c) * 2;
                const durH = Math.min(_getDurH(), maxDurH);
                const dur  = Math.min(Math.ceil(durH / 2), _colsPerMeasure() - c);
                _cells[m][c][i] = { dur, durH };
              }
            }
            _onGridChange();
          });
```

Reemplázalo por:

```js
          cell.addEventListener('click', (e) => {
            e.preventDefault();
            const m = +cell.dataset.m, c = +cell.dataset.c, i = +cell.dataset.i;
            const raw = _cells[m][c][i];
            if (raw && typeof raw === 'object') {
              if (_curArt || _curOrn) {
                // A modifier toggle is active → apply/remove it on the existing note
                if (_curArt) raw.art = (raw.art === _curArt) ? undefined : _curArt;
                if (_curOrn) raw.orn = (raw.orn === _curOrn) ? undefined : _curOrn;
              } else {
                // Toggle off an existing note
                _cells[m][c][i] = null;
              }
            } else {
              // If a previous long note covers this cell, clear that source note instead
              const occupant = _getOccupantCell(m, c, i);
              if (occupant) {
                _cells[occupant.m][occupant.c][i] = null;
              } else if (raw === null) {
                // Place new note; clamp to measure boundary
                const maxDurH = (_colsPerMeasure() - c) * 2;
                const durH = Math.min(_getDurH(), maxDurH);
                const dur  = Math.min(Math.ceil(durH / 2), _colsPerMeasure() - c);
                const note = { dur, durH };
                if (_curArt) note.art = _curArt;
                if (_curOrn) note.orn = _curOrn;
                _cells[m][c][i] = note;
              }
            }
            _onGridChange();
          });
```

- [ ] **Step 2: Verificar adjuntado al colocar (vía API)**

En `preview_eval`, simula tener `accent` activo y colocar: como aún no hay botones (Task 3), fuerza el estado interno con un click programático no es posible; en su lugar verifica la rama existente vía API:

```js
// Coloca una nota normal, luego confirma que applying art on existing works through setState round-trip:
DrumGridModule.setState({ measures:1, subdiv:8, timeSig:{num:4,den:4},
  cells:[Array.from({length:8},()=>Array(10).fill(null))] });
const st = DrumGridModule.getState();
st.cells[0][0][7] = { dur:1, durH:2, art:'accent', orn:'flam' };
DrumGridModule.setState(st);
StateModule.get('abcCode'); // debe contener {/c}!accent!c
```

Expected: el ABC contiene `{/c}!accent!c`, confirmando que celdas con modificadores sobreviven el round-trip de estado.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: attach articulation/ornament to placed and existing notes

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```

---

## Task 3: Panel de selección — botones toggle + handlers + fix de selector

**Files:**
- Modify: `index.html` (HTML del panel ~879-886; selector del duration picker ~1621-1623; handlers nuevos tras ~1631)

- [ ] **Step 1: Añadir los grupos de botones al HTML**

Localiza el grupo "Figura" y su cierre (≈879-886):

```html
          <div class="gc-group">
            <span class="gc-label">Figura</span>
            <button class="dur-btn" data-frac16="1" title="Semicorchea (1/16) — requiere subdivisión ×16">♬</button>
            <button class="dur-btn active" data-frac16="2" title="Corchea (1/8)">♪</button>
            <button class="dur-btn" data-frac16="4" title="Negra (1/4)">♩</button>
            <button class="dur-btn" data-frac16="8" title="Blanca (1/2)">½</button>
            <button id="btn-dot-toggle" title="Puntillo — activa para añadir puntillo a la figura">·</button>
          </div>
```

Inserta INMEDIATAMENTE DESPUÉS de ese `</div>` de cierre:

```html
          <div class="gc-group">
            <span class="gc-label">Articulación</span>
            <button id="btn-art-accent" class="dur-btn" title="Acento (nota más fuerte)">&gt;</button>
            <button id="btn-art-ghost" class="dur-btn" title="Ghost note (nota fantasma, más suave)">( )</button>
          </div>
          <div class="gc-group">
            <span class="gc-label">Ornamento</span>
            <button id="btn-orn-flam" class="dur-btn" title="Flam (una nota de gracia)">Flam</button>
            <button id="btn-orn-drag" class="dur-btn" title="Drag (dos notas de gracia)">Drag</button>
            <button id="btn-orn-ruff" class="dur-btn" title="Ruff (tres notas de gracia)">Ruff</button>
            <button id="btn-orn-roll" class="dur-btn" title="Roll (redoble / trémolo)">Roll</button>
          </div>
```

- [ ] **Step 2: Acotar el handler del duration picker para que no colisione**

Los botones nuevos comparten la clase `.dur-btn` (para heredar estilo) pero NO deben formar parte del selector de duración. Localiza (≈1621-1626):

```js
          /* Duration picker */
          document.querySelectorAll('.dur-btn').forEach(btn => {
            btn.addEventListener('click', () => {
              document.querySelectorAll('.dur-btn').forEach(b => b.classList.remove('active'));
              btn.classList.add('active');
              _curDurFrac16 = parseInt(btn.dataset.frac16, 10);
            });
          });
```

Reemplázalo por (selector restringido a los que tienen `data-frac16`):

```js
          /* Duration picker — only buttons with data-frac16 belong here */
          document.querySelectorAll('.dur-btn[data-frac16]').forEach(btn => {
            btn.addEventListener('click', () => {
              document.querySelectorAll('.dur-btn[data-frac16]').forEach(b => b.classList.remove('active'));
              btn.classList.add('active');
              _curDurFrac16 = parseInt(btn.dataset.frac16, 10);
            });
          });
```

- [ ] **Step 3: Añadir los handlers de articulación y ornamento**

Localiza el handler del puntillo (≈1628-1631):

```js
          document.getElementById('btn-dot-toggle').addEventListener('click', () => {
            _isDotted = !_isDotted;
            document.getElementById('btn-dot-toggle').classList.toggle('active', _isDotted);
          });
```

Inserta INMEDIATAMENTE DESPUÉS de ese bloque:

```js
          /* Articulation toggles — accent and ghost are mutually exclusive */
          function _setArt(val) {
            _curArt = (_curArt === val) ? null : val;
            document.getElementById('btn-art-accent').classList.toggle('active', _curArt === 'accent');
            document.getElementById('btn-art-ghost').classList.toggle('active', _curArt === 'ghost');
          }
          document.getElementById('btn-art-accent').addEventListener('click', () => _setArt('accent'));
          document.getElementById('btn-art-ghost').addEventListener('click', () => _setArt('ghost'));

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

- [ ] **Step 4: Verificar botones e interacción end-to-end**

Recarga el preview. Con `preview_click`:
1. Click en `#btn-art-accent` → confirma clase `active` (vía `preview_eval`: `document.getElementById('btn-art-accent').classList.contains('active')` → `true`).
2. Click en `#btn-art-ghost` → accent pierde `active`, ghost gana `active` (mutuamente excluyentes).
3. Click en una celda de la fila SN del grid → `preview_eval`: `StateModule.get('abcCode')` debe contener `!(.!!).!c`.
4. Click en `#btn-orn-flam`, luego otra celda SN → el ABC debe contener `{/c}`.
5. Click en un botón de duración (`♩`) → `preview_eval`: confirma que `#btn-art-ghost` SIGUE con `active` (el fix de selector evita que el duration picker lo limpie).

Expected: todos pasan. Toma `preview_screenshot` del panel mostrando los grupos nuevos.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat: articulation/ornament toggle buttons in grid panel

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```

---

## Task 4: Indicadores visuales en el grid

**Files:**
- Modify: `index.html` (CSS cerca de `.dg-cell` ~684-702; render de celda en `_renderTable` ~1519-1530)

- [ ] **Step 1: Asegurar posicionamiento relativo y añadir CSS de modificadores**

Localiza la regla base `.dg-cell` (la que termina ≈688-689 con `vertical-align: middle;`) y la regla de nota (≈693-702):

```css
    .dg-cell.note::after {
      content: '';
      display: block;
      width: 14px;
      height: 14px;
      border-radius: 50%;
      background: var(--accent);
      margin: 0 auto;
    }
```

Inserta INMEDIATAMENTE DESPUÉS de esa regla:

```css
    /* Modifier indicators */
    .dg-cell { position: relative; }
    .dg-cell.art-accent::after { box-shadow: 0 0 0 2px var(--accent); }
    .dg-cell.art-ghost::after  { background: transparent; border: 2px solid var(--accent); }
    .orn-tag {
      position: absolute; top: 1px; right: 2px;
      font-size: 8px; line-height: 1; color: var(--accent);
      pointer-events: none;
    }
```

(Si `.dg-cell` ya tuviera `position: relative`, omite esa línea; el resto se mantiene.)

- [ ] **Step 2: Renderizar clases y badge en la celda**

Localiza el cierre del bucle de celdas en `_renderTable` (≈1519-1530):

```js
              const raw       = _cells[m]?.[c]?.[i] ?? null;
              const isNote    = raw && typeof raw === 'object';
              const isCont    = !raw && !!_getOccupantCell(m, c, i);
              const beatStart = _isBeatStart(c);
              const measStart = c === 0;
              let cls = 'dg-cell';
              if (isNote)    cls += ' note';
              else if (raw === 'rest') cls += ' rest';
              else if (isCont)         cls += ' cont';
              if (beatStart) cls += ' beat-start';
              if (measStart) cls += ' measure-start';
              body += `<td class="${cls}" data-m="${m}" data-c="${c}" data-i="${i}"></td>`;
```

Reemplázalo por:

```js
              const raw       = _cells[m]?.[c]?.[i] ?? null;
              const isNote    = raw && typeof raw === 'object';
              const isCont    = !raw && !!_getOccupantCell(m, c, i);
              const beatStart = _isBeatStart(c);
              const measStart = c === 0;
              let cls = 'dg-cell';
              if (isNote)    cls += ' note';
              else if (raw === 'rest') cls += ' rest';
              else if (isCont)         cls += ' cont';
              if (beatStart) cls += ' beat-start';
              if (measStart) cls += ' measure-start';
              let inner = '';
              if (isNote) {
                if (raw.art === 'accent')     cls += ' art-accent';
                else if (raw.art === 'ghost') cls += ' art-ghost';
                if (raw.orn) {
                  const tag = ({ flam: 'F', drag: 'D', ruff: 'Rf', roll: 'Ro' })[raw.orn] || '';
                  inner = `<span class="orn-tag">${tag}</span>`;
                }
              }
              body += `<td class="${cls}" data-m="${m}" data-c="${c}" data-i="${i}">${inner}</td>`;
```

- [ ] **Step 3: Verificar indicadores visuales**

Recarga el preview. Vía `preview_eval` coloca notas con cada modificador:

```js
DrumGridModule.setState({ measures:1, subdiv:8, timeSig:{num:4,den:4}, cells:[[
  [null,null,null,null,null,null,null,{dur:1,durH:2,art:'accent'},null,null],
  [null,null,null,null,null,null,null,{dur:1,durH:2,art:'ghost'},null,null],
  [null,null,null,null,null,null,null,{dur:1,durH:2,orn:'flam'},null,null],
  [null,null,null,null,null,null,null,{dur:1,durH:2,orn:'roll'},null,null],
  ...Array.from({length:4},()=>Array(10).fill(null))
]]});
```

Toma `preview_screenshot` del grid. Expected: la celda con accent muestra un punto con anillo; ghost muestra un punto hueco (solo borde); flam muestra badge "F"; roll muestra badge "Ro".

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: visual indicators for articulations/ornaments in grid

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```

---

## Task 5: Formato shorthand — vocabulario nuevo + objeto `mods`

**Files:**
- Modify: `index.html` (`_parseShorthand` ~1858-1883)

- [ ] **Step 1: Ampliar el parser shorthand**

Localiza `_parseShorthand` completo (≈1858-1883):

```js
      // Convert tutor shorthand → full state.
      // Shorthand: { format:"shorthand", timeSig:"4/4", subdiv:8, measures:1,
      //              tracks:{ HH:"x.x.x.x.", SN:"..x...x.", KI:"x...x..." } }
      // Each char = one grid column; 'x' = note, '.' = empty.
      function _parseShorthand(data) {
        const ABBR = { CR:0, RD:1, RB:2, HH:3, HO:4, T1:5, T2:6, SN:7, TF:8, KI:9 };
        const tsStr  = typeof data.timeSig === 'string' ? data.timeSig
                       : `${data.timeSig.num}/${data.timeSig.den}`;
        const [n, d] = tsStr.split('/').map(Number);
        const timeSig = { num: n, den: d };
        const subdiv   = data.subdiv  || 8;
        const measures = data.measures || 1;
        const cols = Math.round(n * subdiv / d);
        const cells = Array.from({ length: measures }, () =>
          Array.from({ length: cols }, () => Array(10).fill(null)));
        for (const [abbr, pat] of Object.entries(data.tracks || {})) {
          const idx = ABBR[abbr.toUpperCase()];
          if (idx === undefined) continue;
          for (let pos = 0; pos < Math.min(pat.length, measures * cols); pos++) {
            if (pat[pos].toLowerCase() === 'x') {
              cells[Math.floor(pos / cols)][pos % cols][idx] = { dur: 1, durH: 2 };
            }
          }
        }
        return { measures, subdiv, timeSig, cells };
      }
```

Reemplázalo por:

```js
      // Convert tutor shorthand → full state.
      // Shorthand: { format:"shorthand", timeSig:"4/4", subdiv:8, measures:1,
      //              tracks:{ HH:"x.x.x.x.", SN:"g.X.x.f.", KI:"x...x..." },
      //              mods:{ SN:{ "0":{art:"accent",orn:"flam"} } } }
      // Per-position chars: '.'=silence  'x'=note  'X'=accent  'g'=ghost
      //                     'f'=flam  'd'=drag  'r'=roll   (ruff via mods)
      function _parseShorthand(data) {
        const ABBR = { CR:0, RD:1, RB:2, HH:3, HO:4, T1:5, T2:6, SN:7, TF:8, KI:9 };
        // char → cell modifiers (null value = plain note, no modifier)
        const CHAR_MOD = {
          'x': {}, 'X': { art: 'accent' }, 'g': { art: 'ghost' },
          'f': { orn: 'flam' }, 'd': { orn: 'drag' }, 'r': { orn: 'roll' }
        };
        const tsStr  = typeof data.timeSig === 'string' ? data.timeSig
                       : `${data.timeSig.num}/${data.timeSig.den}`;
        const [n, d] = tsStr.split('/').map(Number);
        const timeSig = { num: n, den: d };
        const subdiv   = data.subdiv  || 8;
        const measures = data.measures || 1;
        const cols = Math.round(n * subdiv / d);
        const cells = Array.from({ length: measures }, () =>
          Array.from({ length: cols }, () => Array(10).fill(null)));
        for (const [abbr, pat] of Object.entries(data.tracks || {})) {
          const idx = ABBR[abbr.toUpperCase()];
          if (idx === undefined) continue;
          for (let pos = 0; pos < Math.min(pat.length, measures * cols); pos++) {
            const ch = pat[pos];
            if (ch === '.' || ch === ' ') continue;
            const mod = CHAR_MOD[ch] ?? CHAR_MOD[ch.toLowerCase()];
            if (mod === undefined) {
              throw new Error(`Carácter desconocido '${ch}' en pista ${abbr}, posición ${pos}`);
            }
            cells[Math.floor(pos / cols)][pos % cols][idx] = { dur: 1, durH: 2, ...mod };
          }
        }
        // Optional `mods` overrides: mods[ABBR][colIndex] = { art?, orn? }
        for (const [abbr, byCol] of Object.entries(data.mods || {})) {
          const idx = ABBR[abbr.toUpperCase()];
          if (idx === undefined) continue;
          for (const [colStr, ov] of Object.entries(byCol)) {
            const pos = parseInt(colStr, 10);
            if (!Number.isInteger(pos) || pos < 0 || pos >= measures * cols) continue;
            const cell = cells[Math.floor(pos / cols)][pos % cols][idx];
            if (!cell) {
              throw new Error(`mods en ${abbr} col ${pos} no tiene nota base`);
            }
            if (ov.art) cell.art = ov.art;
            if (ov.orn) cell.orn = ov.orn;
          }
        }
        return { measures, subdiv, timeSig, cells };
      }
```

- [ ] **Step 2: Verificar el parser con vocabulario y `mods`**

En `preview_eval`:

```js
ScoreLibraryModule.importJSON(JSON.stringify({
  name: 'Test Articulaciones', format: 'shorthand',
  timeSig: '4/4', subdiv: 8, measures: 1,
  tracks: { SN: 'g.X.x.f.', KI: 'x...x...' },
  mods:   { SN: { '0': { art: 'accent', orn: 'flam' }, '4': { orn: 'ruff' } } }
}));
StateModule.get('abcCode');
```

Expected: la importación no lanza error; el ABC resultante contiene `{/c}!accent!c` (col 0 con override), `!accent!c` (col 2 → 'X'), `{/c}c` (col 6 → 'f') y `{ccc}c` (col 4 → ruff vía mods). Toma `preview_screenshot` del score.

También prueba el caso de error:

```js
try { ScoreLibraryModule.importJSON(JSON.stringify({
  name:'bad', format:'shorthand', timeSig:'4/4', subdiv:8, measures:1,
  tracks:{ SN:'x.q.' } })); 'NO ERROR'; }
catch(e){ e.message; }   // debe devolver el mensaje "Carácter desconocido 'q'..."
```

Expected: devuelve el mensaje de error de carácter desconocido.

- [ ] **Step 3: Verificar persistencia (round-trip de biblioteca)**

```js
// Tras importar 'Test Articulaciones', recarga desde localStorage:
const saved = JSON.parse(localStorage.getItem('drum-portal-scores'));
const last = saved[saved.length - 1];
JSON.stringify(last.state.cells[0][0][7]);  // debe incluir art:"accent" y orn:"flam"
```

Expected: la celda guardada conserva `art` y `orn`, confirmando que biblioteca/exportar/compartir soportan modificadores sin cambios extra.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: shorthand vocabulary + mods for articulations/ornaments

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```

---

## Verificación final (tras todas las tareas)

1. `preview_screenshot` de un groove completo que combine: HH normal, SN con ghost+accent+flam, KI normal — confirmar que score y grid coinciden.
2. Reproducir con ▶ y confirmar (vía `preview_logs`/audio) que el acento suena más fuerte. (Ghost soft y roll audio son best-effort por limitaciones de abcjs; si no suenan distinto, es aceptable y se documenta.)
3. Confirmar que un score viejo guardado (sin modificadores) sigue cargando idéntico.

## Notas de integración

- Tras completar Fase 1, actualizar el `SKILL.md` del tutor (`...\skills\tutor-de-bateria-avanzado\SKILL.md`) con el vocabulario shorthand nuevo (`X g f d r` + objeto `mods`) para que pueda transcribir las figuras de las fotos. (Edición de docs, no de código — hacerla al cerrar la fase.)
- Fase 2 (tresillos/grupillos) tendrá su propio spec y plan.
