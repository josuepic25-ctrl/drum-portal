# Tresillos / Subdivisión Mixta por Beat — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Permitir que un mismo compás mezcle beats rectos y beats en tresillo, con un modo de escritura tresillo que no transforma lo ya escrito, reproducible y notado correctamente (incluyendo rolls dentro de tresillos).

**Architecture:** El modelo de celdas pasa de una matriz rectangular `_cells[m][col][inst]` a una lista de grupos-por-beat `_cells[m] = [{kind, div, cells}]`. Cada beat decide su subdivisión. La emisión ABC procesa beat por beat (cada beat es autocontenido — **las notas se acotan al beat**, no cruzan beats); los beats tresillo se envuelven en un tuplet explícito `(p:q:r` donde `r` cuenta las notas reales emitidas (incluidos los golpes de un roll). El motor de rolls (`_expandRoll`) no se toca — el espaciado sigue derivándose de `durX`, así que un roll dentro de un tresillo se escala con el corchete.

**Tech Stack:** Single-file `index.html`, JS vanilla (IIFEs), abcjs v6.4.4 (render SVG + MIDI), notación ABC `L:1/32` con duraciones enteras. Verificación vía `preview_*` (servir el dir y evaluar `DrumGridModule.getABC()` / `ScoreLibraryModule.importJSON()` en el contexto de página).

**Decisión de alcance (acotar al beat):** Los tresillos rompen la línea de tiempo de enteros de 1/32 (un tresillo de corchea = 8/3 unidades). Por eso cada beat se emite como unidad autocontenida y **una nota no puede cruzar el límite de su beat** (se acota a `div - c` celdas). Esto cambia el comportamiento de notas largas que hoy cruzan beats dentro de un compás; los grooves del tutor usan notas de 1 celda, así que no se ven afectados. Atar notas entre beats queda fuera de alcance (aditivo a futuro).

---

## Estructura de archivos

- **Modificar:** `C:\Users\JosuReborn\OneDrive\Documentos\Proyectos_Arquitectura\index.html` — todo el trabajo de modelo, render, emisión, UI y shorthand.
- **Modificar:** `C:\Users\JosuReborn\AppData\Roaming\Claude\local-agent-mode-sessions\skills-plugin\b11912fc-1893-424f-8615-dd0734550cf4\2a2d206d-faf2-49e4-8b24-7ea3b5965f5a\skills\tutor-de-bateria-avanzado\SKILL.md` — documentar `beats` + separador `|` + guía de transcripción.

## Cómo verificar (patrón común a todas las tareas)

El verificador (controlador) sirve el directorio y evalúa en el contexto de la página. Helper de aserción que se usa en los pasos:

```js
// preview_eval — devuelve el ABC actual del grid
() => DrumGridModule.getABC()

// preview_eval — importa un shorthand y devuelve el ABC resultante
(json) => { ScoreLibraryModule.importJSON(JSON.stringify(json)); return DrumGridModule.getABC(); }
```

Las aserciones se hacen sobre el **string ABC** (subcadenas, conteos de notas por instrumento) y sobre el **DOM** del grid (`document.querySelectorAll('#drum-grid .dg-cell')`). Los tres símbolos `DrumGridModule`, `ScoreLibraryModule` y `RendererModule` son `const` de ámbito de script y resuelven desde `eval` global (igual que en la Fase 1).

---

## Task 1: Refactor del modelo a grupos-por-beat (recto + capaz de tresillo)

Cambia el almacenamiento de celdas, los helpers, la migración, el render, los handlers de clic y la emisión ABC, todo a la vez (el cambio de forma es atómico). Al terminar: los grooves rectos se ven y suenan **idénticos**, y un beat tresillo inyectado a mano ya emite un tuplet y se dibuja con sus 3/6 celdas.

**Files:**
- Modify: `index.html` — `_getOccupantCell`, `_initCells`, `_colLabel`, `_isBeatStart`, `_buildABC` (rewrite de `buildVoice`), `_renderTable`, handlers click/contextmenu, `_migrateCells`, comentario del modelo y helpers nuevos.

- [ ] **Step 1: Añadir helpers de subdivisión por beat**

En `DrumGridModule`, justo después de `_getDur()` (línea ~1419), añade:

```js
      // ── Subdivisión por beat (Fase tresillos) ──
      // Un beat = una unidad del denominador. beatsPerMeasure = num.
      function _beatsPerMeasure() { return _timeSig.num; }
      // Celdas de un beat recto = colsPerBeat = subdiv/den (2 en ×8, 4 en ×16 para /4).
      function _straightDiv()   { return Math.round(_subdiv / _timeSig.den); }
      // Celdas de un beat tresillo = recto × 3/2 (3 en ×8, 6 en ×16).
      function _tripletDiv()    { return Math.round(_straightDiv() * 3 / 2); }
      // Crea un grupo de beat vacío del tipo indicado.
      function _makeBeat(kind) {
        const div = kind === 'triplet' ? _tripletDiv() : _straightDiv();
        return {
          kind,
          div,
          cells: Array.from({ length: div }, () => new Array(INSTRUMENTS.length).fill(null))
        };
      }
```

- [ ] **Step 2: Reescribir `_initCells` para crear grupos-por-beat**

Reemplaza `_initCells` (líneas ~1442-1447) por:

```js
      function _initCells() {
        const beats = _beatsPerMeasure();
        _cells = Array.from({ length: _measures }, () =>
          Array.from({ length: beats }, () => _makeBeat('straight'))
        );
      }
```

- [ ] **Step 3: Reescribir `_getOccupantCell` para que sea de ámbito beat**

Con notas acotadas al beat, la mirada-atrás solo recorre celdas del mismo beat. Reemplaza `_getOccupantCell` (líneas ~1422-1440) por:

```js
      // Devuelve {c} de la celda que "ocupa" la posición (m,b,c,i) por su duración,
      // o null si está libre. Solo mira dentro del mismo beat (las notas no cruzan beats).
      function _getOccupantCell(m, b, c, i) {
        const group = _cells[m]?.[b];
        if (!group) return null;
        for (let pc = c - 1; pc >= 0; pc--) {
          const cell = group.cells[pc]?.[i];
          if (cell && typeof cell === 'object') {
            return (c - pc) < cell.dur ? { c: pc } : null;
          }
          if (cell === 'rest') break;
        }
        return null;
      }
```

- [ ] **Step 4: Reescribir `_colLabel` / `_isBeatStart` para índices por beat**

Estas ya no reciben una columna global de compás sino el índice de celda dentro de un beat (`c`) y el `div` del beat. Reemplaza ambas (líneas ~1449-1463) por:

```js
      /* Etiqueta de una celda dentro de un beat (0-indexada). c==0 → número de beat. */
      function _cellLabel(beatNum, c, div, kind) {
        if (c === 0) return String(beatNum);
        if (kind === 'triplet') return div === 3 ? ['', 'tr', 'let'][c] : '';
        const SUB = { 2: ['', '+'], 4: ['', 'e', '+', 'a'] };
        return (SUB[div] || [])[c] || '';
      }
```

(Se elimina `_isBeatStart`; el inicio de beat ahora es siempre `c === 0`.)

- [ ] **Step 5: Añadir migración de formato rectangular en `_migrateCells`**

Reemplaza `_migrateCells` (líneas ~1711-1726) por una versión que primero migra el `orn:'roll'` heredado y luego convierte la forma rectangular vieja a grupos-por-beat:

```js
      // Migra celdas antiguas. (1) orn:'roll' (string) → roll:{type:'buzz'}.
      // (2) Forma rectangular vieja _cells[m][col][inst] → grupos-por-beat
      //     _cells[m] = [{kind:'straight', div, cells}]. Idempotente.
      function _migrateCells(cells) {
        if (!Array.isArray(cells)) return cells;

        // (1) orn:'roll' heredado — recorre cualquier celda-objeto que encuentre.
        const fixRoll = (cell) => {
          if (cell && typeof cell === 'object' && cell.orn === 'roll') {
            delete cell.orn;
            cell.roll = { type: 'buzz', accentRelease: true };
          }
        };

        // Detección de formato viejo: _cells[m] es un array cuyo primer elemento
        // NO tiene la forma {kind, div, cells}. (Un grupo-por-beat sí la tiene.)
        const isOldRect = cells.length > 0 && Array.isArray(cells[0]) &&
          !(cells[0][0] && typeof cells[0][0] === 'object' &&
            'kind' in cells[0][0] && 'cells' in cells[0][0]);

        if (isOldRect) {
          const beats = _beatsPerMeasure();
          const div   = _straightDiv();
          const out = [];
          for (const meas of cells) {
            const cols = Array.isArray(meas) ? meas.length : 0;
            const colsPerBeat = beats > 0 ? Math.round(cols / beats) : cols;
            const measOut = [];
            for (let b = 0; b < beats; b++) {
              const groupCells = [];
              for (let c = 0; c < colsPerBeat; c++) {
                const srcCol = meas[b * colsPerBeat + c];
                const arr = Array.isArray(srcCol)
                  ? srcCol.map(cell => { fixRoll(cell); return cell; })
                  : new Array(INSTRUMENTS.length).fill(null);
                groupCells.push(arr);
              }
              measOut.push({ kind: 'straight', div: colsPerBeat || div, cells: groupCells });
            }
            out.push(measOut);
          }
          return out;
        }

        // Formato nuevo: solo aplica fixRoll dentro de cada grupo.
        for (const meas of cells) {
          if (!Array.isArray(meas)) continue;
          for (const group of meas) {
            if (!group || !Array.isArray(group.cells)) continue;
            for (const col of group.cells) {
              if (Array.isArray(col)) for (const cell of col) fixRoll(cell);
            }
          }
        }
        return cells;
      }
```

- [ ] **Step 6: Reescribir la emisión ABC (`buildVoice` → emisión beat-por-beat)**

Dentro de `_buildABC`, reemplaza toda la función `buildVoice` (líneas ~1507-1599) por la versión beat-por-beat siguiente. `cellIn32`, `halfLIn32`, `REPR32`, `fmtLen`, `emitRest`, `INSTRUMENTS`, `_expandRoll`, `ROLL_TYPES` y `_timeSig` ya están en ámbito.

```js
        // Emite el contenido ABC de un beat (autocontenido). Devuelve string.
        // Beats tresillo se envuelven en tuplet explícito (div:q:r con r = nº de
        // notas/silencios emitidos, contando los golpes de los rolls).
        function emitBeat(group, idxs) {
          const div       = group.div;
          const faceUnit  = cellIn32;            // unidades 1/32 de "valor escrito" por celda
          const beatFace  = div * faceUnit;      // total de unidades-cara en el beat
          const realUnits = Math.round(32 / _timeSig.den); // duración real del beat (1/32)

          // 1) Recolectar eventos-acorde dentro del beat
          const nextFreeAt = {};
          idxs.forEach(i => nextFreeAt[i] = 0);  // en celdas
          const events = [];
          for (let c = 0; c < div; c++) {
            const posF = c * faceUnit;
            const notesHere = [];
            for (const i of idxs) {
              if (nextFreeAt[i] <= c) {
                const cell = group.cells[c]?.[i];
                if (cell && typeof cell === 'object') {
                  const durH = cell.durH ?? cell.dur * 2;
                  const durX = durH * halfLIn32;
                  notesHere.push({ note: INSTRUMENTS[i].note, dur: cell.dur, durX, art: cell.art, orn: cell.orn, roll: cell.roll });
                }
              }
            }
            if (notesHere.length > 0) {
              const dur  = Math.max(...notesHere.map(n => n.dur));
              const durX = Math.max(...notesHere.map(n => n.durX));
              for (const i of idxs) {
                if (nextFreeAt[i] <= c && group.cells[c]?.[i] && typeof group.cells[c][i] === 'object') {
                  nextFreeAt[i] = c + dur;
                }
              }
              const art    = notesHere.find(n => n.art)?.art || null;
              const ornEv  = notesHere.find(n => n.orn);
              const orn    = ornEv?.orn || null;
              const rollEv = notesHere.find(n => n.roll);
              const roll   = rollEv?.roll || null;
              const graceNote = ornEv ? ornEv.note : notesHere[0].note;
              events.push({ posF, notes: notesHere.map(n => n.note), dur, durX, art, orn, roll, graceNote });
            }
          }

          // Beat vacío → un silencio de la duración real del beat, sin tuplet.
          if (events.length === 0) return emitRest(realUnits);

          // 2) Emitir contenido sobre [0, beatFace), rellenando huecos; contar tokens
          let inner = '';
          let count = 0;
          let posF  = 0;
          let evIdx = 0;
          const restCounted = (n) => {
            while (n > 0) {
              const best = REPR32.find(r => r <= n) || 1;
              inner += 'z' + fmtLen(best);
              count++;
              n -= best;
            }
          };
          while (posF < beatFace) {
            const ev = evIdx < events.length ? events[evIdx] : null;
            if (ev && ev.posF === posF) {
              evIdx++;
              let prefix = '';
              const g = ev.graceNote;
              if (ev.orn === 'flam')      prefix += `{/${g}}`;
              else if (ev.orn === 'drag') prefix += `{${g}${g}}`;
              else if (ev.orn === 'ruff') prefix += `{${g}${g}${g}}`;
              let deco = '', decoClose = '';
              if (ev.art === 'accent')     deco += '!accent!';
              else if (ev.art === 'ghost') { deco += '!(.!'; decoClose = '!).!'; }
              const chord = ev.notes.length === 1 ? ev.notes[0] : '[' + ev.notes.join('') + ']';
              const durX  = Math.min(ev.durX, beatFace - posF); // acotar al beat
              if (ev.roll) {
                const sizes = _expandRoll(durX, ev.roll.type);
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
                  inner += pre + dOpen + acc + trem + chord + fmtLen(sizes[h]) + dClose;
                  count++;
                }
              } else {
                inner += prefix + deco + chord + fmtLen(durX) + decoClose;
                count++;
              }
              posF += durX;
            } else if (ev && ev.posF > posF) {
              restCounted(ev.posF - posF);
              posF = ev.posF;
            } else {
              restCounted(beatFace - posF);
              posF = beatFace;
            }
          }

          // 3) Envolver beats tresillo en tuplet explícito
          if (group.kind === 'triplet') {
            const q = Math.round(div * 2 / 3);   // 3→2, 6→4 (ratio 2/3)
            return `(${div}:${q}:${count}` + inner;
          }
          return inner;
        }

        // Construye la secuencia ABC para un subconjunto de instrumentos.
        function buildVoice(idxs) {
          let voice = '';
          for (let m = 0; m < _measures; m++) {
            voice += '|';
            const beats = _cells[m] || [];
            for (let b = 0; b < beats.length; b++) {
              voice += emitBeat(beats[b], idxs);
            }
          }
          voice += '|\n';
          return voice;
        }
```

Elimina del cuerpo de `_buildABC` las líneas ahora muertas que dependían de la forma rectangular: la constante `const cols = _colsPerMeasure();` al inicio de `_buildABC` (línea ~1466) ya no se usa dentro de `buildVoice` — déjala solo si algo más la usa; si no, elimínala.

- [ ] **Step 7: Reescribir `_renderTable` para iterar beats**

Reemplaza `_renderTable` (líneas ~1606-1707) por la versión que recorre measures→beats→cells. Mantiene los `data-m`/`data-i` y añade `data-b`/`data-c`; marca los beats tresillo con la clase `triplet-beat` en su cabecera.

```js
      function _renderTable() {
        if (!_tableEl) return;

        // Lista plana de columnas a renderizar, en orden, con metadatos.
        const flat = [];   // { m, b, c, div, kind, beatNum, beatStart, measStart }
        let beatNum = 0;
        for (let m = 0; m < _measures; m++) {
          const beats = _cells[m] || [];
          for (let b = 0; b < beats.length; b++) {
            beatNum++;
            const group = beats[b];
            for (let c = 0; c < group.div; c++) {
              flat.push({
                m, b, c, div: group.div, kind: group.kind, beatNum,
                beatStart: c === 0,
                measStart: b === 0 && c === 0
              });
            }
          }
        }
        const totalCols = flat.length;

        /* Cabecera */
        let hdr = '<thead><tr><th class="dg-row-head"></th>';
        for (const col of flat) {
          let cls = 'dg-col-header';
          if (col.beatStart) cls += ' beat-start';
          if (col.measStart) cls += ' measure-start';
          if (col.kind === 'triplet') cls += ' triplet-beat';
          const label = _cellLabel(col.beatNum, col.c, col.div, col.kind);
          const badge = (col.kind === 'triplet' && col.c === 0) ? '<span class="trip-badge">3</span>' : '';
          hdr += `<th class="${cls}">${badge}${label}</th>`;
        }
        hdr += '</tr></thead>';

        /* Cuerpo */
        let body = '<tbody>';
        for (let i = 0; i < INSTRUMENTS.length; i++) {
          body += `<tr><td class="dg-row-head">${INSTRUMENTS[i].abbr}</td>`;
          for (const col of flat) {
            const group  = _cells[col.m][col.b];
            const raw    = group.cells[col.c]?.[i] ?? null;
            const isNote = raw && typeof raw === 'object';
            const isCont = !raw && !!_getOccupantCell(col.m, col.b, col.c, i);
            let cls = 'dg-cell';
            if (isNote)              cls += ' note';
            else if (raw === 'rest') cls += ' rest';
            else if (isCont)         cls += ' cont';
            if (col.beatStart)       cls += ' beat-start';
            if (col.measStart)       cls += ' measure-start';
            if (col.kind === 'triplet') cls += ' triplet-beat';
            let inner = '';
            if (isNote) {
              if (raw.art === 'accent')     cls += ' art-accent';
              else if (raw.art === 'ghost') cls += ' art-ghost';
              let tag = '';
              if (raw.roll)     tag = (ROLL_TYPES[raw.roll.type] || {}).tag || 'R';
              else if (raw.orn) tag = ({ flam: 'F', drag: 'D', ruff: 'Rf' })[raw.orn] || '';
              if (tag) inner = `<span class="orn-tag">${tag}</span>`;
            }
            body += `<td class="${cls}" data-m="${col.m}" data-b="${col.b}" data-c="${col.c}" data-i="${i}">${inner}</td>`;
          }
          body += '</tr>';
          if (i === 4 || i === 7) {
            body += `<tr class="dg-row-separator"><td colspan="${totalCols + 1}"></td></tr>`;
          }
        }
        body += '</tbody>';

        _tableEl.innerHTML = hdr + body;
        _attachCellListeners();
      }
```

- [ ] **Step 8: Extraer y reescribir los listeners de celda con la nueva direccion**

Añade una función `_attachCellListeners` (justo después de `_renderTable`) que reemplaza el bloque de listeners viejo (líneas ~1664-1706). Usa `data-b`/`data-c` y `group.div` para acotar la duración al beat:

```js
      function _attachCellListeners() {
        _tableEl.querySelectorAll('.dg-cell').forEach(cell => {
          cell.addEventListener('click', (e) => {
            e.preventDefault();
            const m = +cell.dataset.m, b = +cell.dataset.b, c = +cell.dataset.c, i = +cell.dataset.i;
            const group = _cells[m][b];
            const raw = group.cells[c][i];
            if (raw && typeof raw === 'object') {
              if (_curArt || _curOrn || _curRoll) {
                if (_curArt)  raw.art  = (raw.art === _curArt) ? undefined : _curArt;
                if (_curOrn)  raw.orn  = (raw.orn === _curOrn) ? undefined : _curOrn;
                if (_curRoll) raw.roll = (raw.roll && raw.roll.type === _curRoll.type) ? undefined : { ..._curRoll };
              } else {
                group.cells[c][i] = null;
              }
            } else {
              const occupant = _getOccupantCell(m, b, c, i);
              if (occupant) {
                group.cells[occupant.c][i] = null;
              } else if (raw === null) {
                const maxDurH = (group.div - c) * 2;          // acotar al beat
                const durH = Math.min(_getDurH(), maxDurH);
                const dur  = Math.min(Math.ceil(durH / 2), group.div - c);
                const note = { dur, durH };
                if (_curArt)  note.art  = _curArt;
                if (_curOrn)  note.orn  = _curOrn;
                if (_curRoll) note.roll = { ..._curRoll };
                group.cells[c][i] = note;
              }
            }
            _onGridChange();
          });
          cell.addEventListener('contextmenu', (e) => {
            e.preventDefault();
            const m = +cell.dataset.m, b = +cell.dataset.b, c = +cell.dataset.c, i = +cell.dataset.i;
            const group = _cells[m][b];
            const raw = group.cells[c][i];
            group.cells[c][i] = (raw === 'rest') ? null : 'rest';
            _onGridChange();
          });
        });
      }
```

- [ ] **Step 9: Actualizar el comentario del modelo de celdas**

Reemplaza el comentario de la línea ~1403 por:

```js
      // _cells[measure] = [ beatGroup, ... ]   (un grupo por beat del compás)
      // beatGroup = { kind:'straight'|'triplet', div:Number, cells: Cell[div][instIdx] }
      // Cell = null | 'rest' | { dur, durH, art?, orn?, roll? }
```

- [ ] **Step 10: Servir el portal y verificar retrocompatibilidad recta**

Inicia el preview sirviendo el directorio del portal. Evalúa (importando un groove recto clásico y leyendo el ABC):

```js
(() => {
  ScoreLibraryModule.importJSON(JSON.stringify({
    name: 'recto', format: 'shorthand', timeSig: '4/4', subdiv: 8, measures: 1,
    tracks: { HH: 'x.x.x.x.', SN: '..x...x.', KI: 'x...x...' }
  }));
  const abc  = DrumGridModule.getABC();
  // Recorta la cabecera (contiene 'clef=perc') antes de contar notas de snare ('c').
  const body = abc.slice(abc.indexOf('clef=perc') + 9);
  const snareHits = (body.match(/c/g) || []).length;
  return {
    hasTuplet: abc.includes('(3') || abc.includes('(6'),  // debe ser false
    snareHits,                                            // debe ser 2
    bars: (body.match(/\|/g) || []).length                // estructura de barras presente
  };
})()
```

Expected: `{ hasTuplet: false, snareHits: 2, bars: >=2 }`. Toma `preview_snapshot` del grid y confirma que se ve como un compás recto normal (8 columnas de cabecera, sin badges "3").

- [ ] **Step 11: Verificar que un beat tresillo inyectado emite tuplet y se dibuja**

Evalúa (construye un estado a mano con el beat 1 recto y el beat 2 tresillo de 3 celdas, con snare en cada celda del tresillo):

```js
(() => {
  const straight = { kind:'straight', div:2, cells:[ [null,null,null,null,null,null,null,{dur:1,durH:2},null,null], new Array(10).fill(null) ] };
  const trip = { kind:'triplet', div:3, cells:[
    [null,null,null,null,null,null,null,{dur:1,durH:2},null,null],
    [null,null,null,null,null,null,null,{dur:1,durH:2},null,null],
    [null,null,null,null,null,null,null,{dur:1,durH:2},null,null]
  ]};
  DrumGridModule.setState({ measures:1, subdiv:8, timeSig:{num:2,den:4}, cells:[ [straight, trip] ] });
  const abc  = DrumGridModule.getABC();
  const body = abc.slice(abc.indexOf('clef=perc') + 9);
  return {
    hasTriplet: abc.includes('(3:2:3'),        // tuplet 3-en-2 de 3 notas
    snareHits: (body.match(/c/g) || []).length, // 1 (recto) + 3 (tresillo) = 4
    badges: document.querySelectorAll('#drum-grid .trip-badge').length // 1
  };
})()
```

Expected: `{ hasTriplet: true, snareHits: 4, badges: 1 }`. `preview_screenshot` para confirmar que abcjs dibuja el corchete "3" sobre el segundo beat (si no aparece el corchete pero el tuplet está en el ABC, registrar como riesgo de render — el sonido será correcto igual).

- [ ] **Step 12: Commit**

```bash
git add index.html
git commit -m "feat(triplets): per-beat cell model — beat groups, beat-scoped emission + tuplet wrapping

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```

---

## Task 2: Modo tresillo de escritura + UI

Añade el estado `_tripletMode`, el botón toggle (HTML/CSS/wiring), la visualización de beats vacíos según el modo, la fijación del `kind` al colocar la primera nota, y la reconversión de un beat por clic en su cabecera (con confirmación si tiene notas).

**Files:**
- Modify: `index.html` — estado `_tripletMode`, HTML del control (~línea 917), CSS (~línea 707), wiring en `init`, ajuste de `_renderTable` para beats vacíos + cabeceras clicables, handler de colocación.

- [ ] **Step 1: Añadir estado `_tripletMode`**

Tras `let _curRoll = null;` (línea ~1402) añade:

```js
      let _tripletMode = false;  // modo de escritura tresillo (toggle)
```

- [ ] **Step 2: Añadir el botón de modo tresillo al HTML**

Tras el `<div class="gc-group">` de "Subdiv." (cierra en línea ~917), añade un nuevo grupo:

```html
          <div class="gc-group">
            <span class="gc-label">Tresillo</span>
            <button id="btn-triplet-mode" class="dur-btn" title="Modo tresillo: lo que escribas irá en tresillos hasta desactivarlo" aria-pressed="false">3</button>
          </div>
```

- [ ] **Step 3: Añadir CSS para el botón activo, el badge y la cabecera de tresillo**

Tras la regla `.dg-col-header.measure-start` (línea ~708) añade:

```css
    #btn-triplet-mode.active { background: var(--accent); color: #0d1117; border-color: var(--accent); }
    .dg-col-header.triplet-beat { color: var(--accent); }
    .dg-col-header.triplet-beat.beat-start { cursor: pointer; }
    .trip-badge {
      display: inline-block;
      font-size: 0.55rem;
      font-weight: 700;
      color: var(--accent);
      vertical-align: super;
      margin-right: 1px;
    }
    .dg-cell.triplet-beat { background: rgba(var(--accent-rgb), 0.04); }
```

- [ ] **Step 4: Mostrar beats vacíos según el modo + cabeceras de beat clicables**

En `_renderTable`, el bucle que construye `flat` debe usar, para un beat **vacío**, la subdivisión del modo activo (sin mutar el dato). Reemplaza el bucle de construcción de `flat` (Step 7 de Task 1) por:

```js
        const flat = [];
        let beatNum = 0;
        for (let m = 0; m < _measures; m++) {
          const beats = _cells[m] || [];
          for (let b = 0; b < beats.length; b++) {
            beatNum++;
            const group = beats[b];
            const empty = _beatIsEmpty(group);
            // Beat vacío se MUESTRA según el modo activo (no se fija hasta escribir).
            const showKind = empty ? (_tripletMode ? 'triplet' : 'straight') : group.kind;
            const showDiv  = empty ? (_tripletMode ? _tripletDiv() : _straightDiv()) : group.div;
            for (let c = 0; c < showDiv; c++) {
              flat.push({
                m, b, c, div: showDiv, kind: showKind, beatNum,
                beatStart: c === 0,
                measStart: b === 0 && c === 0
              });
            }
          }
        }
```

Añade el helper `_beatIsEmpty` junto a `_makeBeat`:

```js
      function _beatIsEmpty(group) {
        return group.cells.every(col => col.every(cell => cell === null));
      }
```

En la cabecera (`hdr`), haz clicable la celda de inicio de beat para reconvertir. Cambia la línea que arma cada `<th>` para añadir datos:

```js
          const beatAttrs = col.beatStart ? ` data-beat-m="${col.m}" data-beat-b="${col.b}"` : '';
          hdr += `<th class="${cls}"${beatAttrs}>${badge}${label}</th>`;
```

> **Nota de acceso a celdas con beat vacío mostrado en otra subdivisión:** cuando un beat vacío `straight(div=2)` se muestra como `triplet(div=3)`, el cuerpo accede a `_cells[m][b].cells[c]` para `c` hasta `showDiv-1`, que puede exceder `group.div`. Protege el acceso en el cuerpo: usa `const raw = group.cells[col.c]?.[i] ?? null;` (ya está con `?.`), de modo que las celdas "extra" se muestran vacías. Al colocar una nota en ellas, el Step 6 reconstruye el beat al `kind`/`div` mostrado.

- [ ] **Step 5: Wiring del botón de modo tresillo**

En `init`, tras el wiring del botón de puntillo (línea ~1809), añade:

```js
          document.getElementById('btn-triplet-mode').addEventListener('click', () => {
            _tripletMode = !_tripletMode;
            const btn = document.getElementById('btn-triplet-mode');
            btn.classList.toggle('active', _tripletMode);
            btn.setAttribute('aria-pressed', String(_tripletMode));
            _renderTable();   // re-dibuja beats vacíos según el modo
          });
```

- [ ] **Step 6: Fijar el `kind` del beat al colocar la primera nota + reconvertir por cabecera**

En `_attachCellListeners`, dentro del handler de `click`, la rama de "colocar nota nueva" debe **fijar la subdivisión del beat al modo activo si el beat está vacío** antes de colocar. Reemplaza la rama `else if (raw === null)` por:

```js
              } else if (raw === null) {
                // Si el beat está vacío, fíjalo a la subdivisión del modo activo.
                if (_beatIsEmpty(group)) {
                  const wantKind = _tripletMode ? 'triplet' : 'straight';
                  if (group.kind !== wantKind) {
                    _cells[m][b] = _makeBeat(wantKind);
                  }
                }
                const g2 = _cells[m][b];
                const maxDurH = (g2.div - c) * 2;
                const durH = Math.min(_getDurH(), maxDurH);
                const dur  = Math.min(Math.ceil(durH / 2), g2.div - c);
                const note = { dur, durH };
                if (_curArt)  note.art  = _curArt;
                if (_curOrn)  note.orn  = _curOrn;
                if (_curRoll) note.roll = { ..._curRoll };
                g2.cells[c][i] = note;
              }
```

Añade el listener de reconversión de beat al final de `_attachCellListeners` (antes de cerrar la función):

```js
        _tableEl.querySelectorAll('.dg-col-header[data-beat-b]').forEach(th => {
          th.addEventListener('click', () => {
            const m = +th.dataset.beatM, b = +th.dataset.beatB;
            const group = _cells[m][b];
            const empty = _beatIsEmpty(group);
            if (!empty && !confirm('Cambiar la subdivisión de este beat vaciará sus notas. ¿Continuar?')) return;
            _cells[m][b] = _makeBeat(group.kind === 'triplet' ? 'straight' : 'triplet');
            _onGridChange();
          });
        });
```

- [ ] **Step 7: Verificar el modo de escritura y la reconversión**

Sirve el portal. Evalúa la secuencia: activar modo tresillo, comprobar que un beat vacío se muestra en 3 celdas, colocar una nota la fija como tresillo:

```js
(() => {
  DrumGridModule.setState({ measures:1, subdiv:8, timeSig:{num:2,den:4},
    cells:[ [ {kind:'straight',div:2,cells:[new Array(10).fill(null),new Array(10).fill(null)]},
              {kind:'straight',div:2,cells:[new Array(10).fill(null),new Array(10).fill(null)]} ] ] });
  document.getElementById('btn-triplet-mode').click();      // activar modo
  const headerCols = document.querySelectorAll('#drum-grid thead th').length - 1; // sin la esquina
  // beat1 vacío + beat2 vacío, ambos mostrados en tresillo (3+3) = 6
  return { headerColsConModo: headerCols, modoActivo: document.getElementById('btn-triplet-mode').classList.contains('active') };
})()
```

Expected: `{ headerColsConModo: 6, modoActivo: true }`. Luego `preview_click` en una celda de snare del primer beat y verifica con `getABC()` que aparece `(3:2:` (el beat quedó fijado a tresillo). Toma `preview_screenshot`.

- [ ] **Step 8: Commit**

```bash
git add index.html
git commit -m "feat(triplets): triplet write-mode toggle, empty-beat display, per-beat reconvert

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```

---

## Task 3: Shorthand — campo `beats` + separador `|`

Extiende `_parseShorthand` para construir el modelo de grupos-por-beat desde el shorthand, con subdivisión por beat (`beats`) y pistas segmentadas por `|`. Mantiene retrocompatibilidad: cadenas sin `|` se interpretan como rectas con el comportamiento actual.

**Files:**
- Modify: `index.html` — `_parseShorthand` (líneas ~2118-2176).

- [ ] **Step 1: Reescribir `_parseShorthand` con soporte de beats**

Reemplaza `_parseShorthand` completo por:

```js
      function _parseShorthand(data) {
        const ABBR = { CR:0, RD:1, RB:2, HH:3, HO:4, T1:5, T2:6, SN:7, TF:8, KI:9 };
        const CHAR_MOD = {
          'x': {}, 'X': { art: 'accent' }, 'g': { art: 'ghost' },
          'f': { orn: 'flam' }, 'd': { orn: 'drag' },
          'r': { roll: { type: 'buzz', accentRelease: true } }
        };
        const tsStr  = typeof data.timeSig === 'string' ? data.timeSig
                       : `${data.timeSig.num}/${data.timeSig.den}`;
        const [n, d] = tsStr.split('/').map(Number);
        const timeSig = { num: n, den: d };
        const subdiv   = data.subdiv  || 8;
        const measures = data.measures || 1;
        const beatsPerMeasure = n;                       // num
        const straightDiv = Math.round(subdiv / d);      // celdas por beat recto
        const tripletDiv  = Math.round(straightDiv * 3 / 2);
        const totalBeats  = measures * beatsPerMeasure;

        // Subdivisión por beat (índice global). Default: straight.
        const beatKinds = new Array(totalBeats).fill('straight');
        if (data.beats && typeof data.beats === 'object') {
          for (const [k, v] of Object.entries(data.beats)) {
            const bi = parseInt(k, 10);
            if (!Number.isInteger(bi) || bi < 0 || bi >= totalBeats) continue;
            if (v !== 'straight' && v !== 'triplet') {
              throw new Error(`Subdivisión de beat inválida '${v}' en beat ${bi} (usa "straight" o "triplet")`);
            }
            beatKinds[bi] = v;
          }
        }
        const divOf = (bi) => beatKinds[bi] === 'triplet' ? tripletDiv : straightDiv;

        // Construir estructura de grupos-por-beat vacía.
        const cells = [];
        let gbi = 0;
        for (let m = 0; m < measures; m++) {
          const meas = [];
          for (let b = 0; b < beatsPerMeasure; b++) {
            const div = divOf(gbi++);
            meas.push({ kind: beatKinds[gbi - 1], div,
              cells: Array.from({ length: div }, () => new Array(10).fill(null)) });
          }
          cells.push(meas);
        }

        // Colocar notas. Cada pista se segmenta por '|' (un segmento por beat).
        // Retrocompat: si la cadena NO tiene '|', se reparte sobre los beats rectos
        // por longitud fija (comportamiento clásico).
        const setCell = (mIdx, bIdx, cIdx, idx, mod) => {
          const newCell = { dur: 1, durH: 2, ...mod };
          if (newCell.roll) newCell.roll = { ...newCell.roll };
          cells[mIdx][bIdx].cells[cIdx][idx] = newCell;
        };
        for (const [abbr, patRaw] of Object.entries(data.tracks || {})) {
          const idx = ABBR[abbr.toUpperCase()];
          if (idx === undefined) continue;
          let segments;
          if (patRaw.includes('|')) {
            segments = patRaw.split('|');
            if (segments.length !== totalBeats) {
              throw new Error(`Pista ${abbr}: ${segments.length} segmentos '|' pero el compás tiene ${totalBeats} beats`);
            }
          } else {
            // Sin '|': trocear por el div recto (clásico, todo recto).
            segments = [];
            for (let bi = 0; bi < totalBeats; bi++) {
              segments.push(patRaw.slice(bi * straightDiv, (bi + 1) * straightDiv));
            }
          }
          for (let bi = 0; bi < totalBeats; bi++) {
            const seg = segments[bi];
            const expected = divOf(bi);
            if (patRaw.includes('|') && seg.length !== expected) {
              throw new Error(`Pista ${abbr}, beat ${bi}: ${seg.length} caracteres pero se esperaban ${expected} (${beatKinds[bi]})`);
            }
            const mIdx = Math.floor(bi / beatsPerMeasure);
            const bIdx = bi % beatsPerMeasure;
            for (let c = 0; c < Math.min(seg.length, expected); c++) {
              const ch = seg[c];
              if (ch === '.' || ch === ' ') continue;
              const mod = CHAR_MOD[ch] ?? CHAR_MOD[ch.toLowerCase()];
              if (mod === undefined) {
                throw new Error(`Carácter desconocido '${ch}' en pista ${abbr}, beat ${bi}, pos ${c}`);
              }
              setCell(mIdx, bIdx, c, idx, mod);
            }
          }
        }

        // mods: col = índice GLOBAL de columna (recorre beats en orden).
        // Mapea índice global → (m, b, c) recorriendo los divs reales.
        const colMap = [];   // colMap[globalCol] = { m, b, c }
        for (let m = 0; m < measures; m++) {
          for (let b = 0; b < beatsPerMeasure; b++) {
            const gi = m * beatsPerMeasure + b;
            for (let c = 0; c < divOf(gi); c++) colMap.push({ m, b, c });
          }
        }
        for (const [abbr, byCol] of Object.entries(data.mods || {})) {
          const idx = ABBR[abbr.toUpperCase()];
          if (idx === undefined) continue;
          for (const [colStr, ov] of Object.entries(byCol)) {
            const pos = parseInt(colStr, 10);
            const loc = colMap[pos];
            if (!loc) continue;
            const cell = cells[loc.m][loc.b].cells[loc.c][idx];
            if (!cell) {
              throw new Error(`mods en ${abbr} col ${pos} no tiene nota base`);
            }
            if (ov.art) cell.art = ov.art;
            if (ov.orn === 'roll') { cell.roll = { type: 'buzz', accentRelease: true }; delete cell.orn; }
            else if (ov.orn) cell.orn = ov.orn;
            if (ov.roll) {
              if (!ROLL_TYPES[ov.roll.type]) {
                throw new Error(`Tipo de roll desconocido '${ov.roll.type}' en ${abbr} col ${pos}`);
              }
              cell.roll = { type: ov.roll.type, accentRelease: ov.roll.accentRelease !== false };
              delete cell.orn;
            }
          }
        }

        return { measures, subdiv, timeSig, cells };
      }
```

- [ ] **Step 2: Verificar el shorthand mixto recto/tresillo**

Sirve el portal. Evalúa importando un compás 2/4 en `subdiv:8` (beat recto = 2 celdas, beat tresillo = 3 celdas), beat 0 recto, beat 1 tresillo:

```js
(() => {
  ScoreLibraryModule.importJSON(JSON.stringify({
    name: 'mixto', format: 'shorthand', timeSig: '2/4', subdiv: 8, measures: 1,
    beats: { '1': 'triplet' },
    tracks: { SN: 'Xx|xxx' },               // beat0 recto (2) | beat1 tresillo (3)
    mods: { SN: { '0': { art: 'accent' } } }
  }));
  const abc  = DrumGridModule.getABC();
  const body = abc.slice(abc.indexOf('clef=perc') + 9);
  return {
    hasTriplet: abc.includes('(3:2'),            // beat tresillo de 3
    hasAccent: abc.includes('!accent!'),
    snareHits: (body.match(/c/g) || []).length   // beat0: 2 (X,x) + beat1: 3 = 5
  };
})()
```

Expected: `{ hasTriplet: true, hasAccent: true, snareHits: 5 }`. Para el caso sextuplet (`subdiv:16`, beat tresillo = 6 celdas) el segmento debe tener 6 chars (p. ej. `'X..x|xxxxxx'`); un segmento con longitud ≠ `div` esperado debe lanzar el error de validación del Step 1.

- [ ] **Step 3: Verificar retrocompatibilidad del shorthand sin `|`**

```js
(() => {
  ScoreLibraryModule.importJSON(JSON.stringify({
    name: 'clasico', format: 'shorthand', timeSig: '4/4', subdiv: 8, measures: 1,
    tracks: { HH: 'x.x.x.x.', SN: '..x...x.', KI: 'x...x...' }
  }));
  const abc  = DrumGridModule.getABC();
  const body = abc.slice(abc.indexOf('clef=perc') + 9);
  return { hasTuplet: abc.includes('(3') || abc.includes('(6'), snareHits: (body.match(/c/g)||[]).length };
})()
```

Expected: `{ hasTuplet: false, snareHits: 2 }` — idéntico al groove clásico.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat(triplets): shorthand 'beats' field + per-beat '|' segments (backward-compatible)

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```

---

## Task 4: SKILL.md — documentar `beats`, `|` y transcripción de tresillos

**Files:**
- Modify: `...\skills\tutor-de-bateria-avanzado\SKILL.md` — sección de shorthand.

- [ ] **Step 1: Añadir subsección de tresillos tras la tabla de caracteres**

Tras el bloque de `mods` (tras la línea que termina `"4": { "orn": "ruff" } } }`, ~línea 76 del SKILL.md actual), inserta:

````markdown
### Tresillos (subdivisión mixta por beat)

Un compás puede mezclar beats rectos y beats en tresillo. La subdivisión es **por beat**:

- Campo `beats`: `{ "<índiceGlobalDeBeat>": "triplet" }` marca qué beats van en tresillo (0-based, recorre todos los compases). Los no listados son rectos.
- En cada pista, separa los beats con `|`. Cada segmento son las celdas de ese beat:
  - Beat recto: 2 caracteres (`subdiv:8`) o 4 (`subdiv:16`).
  - Beat tresillo: 3 caracteres (`subdiv:8`) o 6 (`subdiv:16`).
- Una cadena **sin** `|` se interpreta como compás recto clásico (retrocompatible).

```json
{
  "name": "Solo 7 — compás mixto",
  "format": "shorthand",
  "timeSig": "2/4",
  "subdiv": 8,
  "measures": 1,
  "beats": { "1": "triplet" },
  "tracks": { "SN": "Xx|xxx" },
  "mods": { "SN": { "0": { "art": "accent" } } }
}
```

Aquí el beat 0 es recto (2 corcheas, la 1ª acentuada) y el beat 1 es un tresillo de corcheas (3 notas).

### Transcribir tresillos desde una foto

Al leer una partitura fotografiada:
1. Identifica el número de beats por compás (numerador del compás).
2. Marca como `triplet` los beats que tengan el **corchete con "3"** sobre las notas.
3. Segmenta cada pista con `|`, un segmento por beat, con la longitud que corresponde a su subdivisión.
4. Combina con la identificación de rolls de siempre (corchete de trémolo + "N str" → `r` o `mods.roll` con el tipo medido `m5`/`m7`/`m9`…).
````

- [ ] **Step 2: Verificar que el ejemplo del SKILL importa sin error**

Sirve el portal. Evalúa importando exactamente el JSON del ejemplo del SKILL:

```js
(() => {
  try {
    ScoreLibraryModule.importJSON(JSON.stringify({
      name:'Solo 7', format:'shorthand', timeSig:'2/4', subdiv:8, measures:1,
      beats:{'1':'triplet'}, tracks:{ SN:'Xx|xxx' }, mods:{ SN:{ '0':{ art:'accent' } } }
    }));
    const abc = DrumGridModule.getABC();
    return { ok:true, hasTriplet: abc.includes('(3:2'), hasAccent: abc.includes('!accent!') };
  } catch(e) { return { ok:false, err:e.message }; }
})()
```

Expected: `{ ok:true, hasTriplet:true, hasAccent:true }`.

- [ ] **Step 3: Commit**

```bash
git add "C:\Users\JosuReborn\AppData\Roaming\Claude\local-agent-mode-sessions\skills-plugin\b11912fc-1893-424f-8615-dd0734550cf4\2a2d206d-faf2-49e4-8b24-7ea3b5965f5a\skills\tutor-de-bateria-avanzado\SKILL.md"
git commit -m "docs(skill): document triplet 'beats' field, '|' segments, photo transcription"
```

---

## Verificación final (controlador, tras todas las tareas)

1. `preview_screenshot` de un compás mixto real del Solo No. 7 (2/4, un beat recto de semicorcheas + un beat tresillo) para confirmar el corchete "3" y la alineación.
2. Reproducir con ▶ y confirmar por audio que el beat tresillo suena 3-en-2 frente al recto.
3. Colocar un roll medido (`m5`) dentro de un beat tresillo y confirmar que el ABC contiene el tuplet con `r` que incluye los golpes del roll, y que suena escalado.
4. Guardar en biblioteca, recargar, confirmar persistencia. Importar un score viejo rectangular y confirmar que migra a beats `straight` sin romperse.
5. `preview_resize` a 380px y confirmar que el grid mixto no desborda.
6. Push a GitHub Pages para que esté en el teléfono.
