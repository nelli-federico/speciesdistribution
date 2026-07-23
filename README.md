# Species Explorer — Handoff tecnico

> **Cos'è questo pacchetto.** NON è un mockup da ricreare: è un'**app funzionante** e
> completa. Il tuo obiettivo con Claude Code è *continuarne lo sviluppo*, testarla
> davvero (browser + dispositivo) e correggerne i bug direttamente nel repo. Fedeltà:
> **produzione** (hi-fi, dati reali live).

---

## 1. Cos'è l'app

App mobile-first di biodiversità: scegli un'**area** su una mappa e vedi le **specie**
osservate lì, con filtri per gruppo tassonomico, periodo e ora del giorno. Ogni specie
ha una scheda (descrizione Wikipedia, stagionalità, mappa osservazioni, evidenze
multi-fonte). UI in **18 lingue**. Dati **live**, nessuna chiave API.

- **Sorgenti dati (tutte gratuite, senza chiave):**
  - iNaturalist API v1 (`/observations/species_counts`, `/taxa`, `/observations`) — conteggi specie, autocomplete, osservazioni.
  - GBIF — conteggi aggiuntivi nella scheda specie (deduplicati vs iNaturalist).
  - Photon (geocoder OSM) — ricerca luoghi (type-ahead).
  - Nominatim (OSM) — confini di un luogo selezionato (1 richiesta).
  - Tile raster OpenStreetMap — mappe Leaflet. ⚠️ Policy OSM vieta uso pesante in
    produzione: se il traffico cresce, migrare a vector tiles (Protomaps/MapTiler). È
    uno swap di ~1 riga in 2 punti (cerca `tile.openstreetmap.org`).

---

## 2. Architettura (LEGGERE PRIMA DI TOCCARE)

L'app è **un unico file**, `index.html`, costruito su un piccolo runtime a componenti
("DC runtime") fornito da `support.js`. **`support.js` è il framework: non modificarlo.**

`index.html` contiene:
- Un `<head>` con meta PWA, manifest, Leaflet (CSS+JS da unpkg), font Google.
- Un template `<x-dc>…</x-dc>`: markup con "buchi" `{{ path }}` e controlli
  `<sc-if>` / `<sc-for>`. **Solo stili inline**, niente CSS a classi (a parte
  `@font-face`/`@keyframes`/reset in `<helmet><style>`).
- Uno `<script type="text/x-dc" data-dc-script>` con **`class Component extends DCLogic`**.

### Come funziona il componente (simil-React class):
- `state = {…}` — tutto lo stato dell'app.
- `renderVals()` — **cuore della UI**: restituisce un oggetto piatto di valori/handler
  che riempiono i `{{ }}` del template. Gira ad ogni render. NON deve lanciare eccezioni
  (vedi §5) e NON deve chiamare `setState`.
- `componentDidMount` / `componentDidUpdate` / `componentWillUnmount` — lifecycle.
  `componentDidUpdate` orchestra: `maybeRefresh`, `maybeGroupCounts`, `syncObsMap`,
  init/re-init mappa, `refreshAreaLayer`, `updatePinPreview`, `syncUrl`. Ogni chiamata
  è **guardata da una "key"** (es. `_lk`, `_gk`, `_areaSig`, `_mapKey`) per non
  rifare lavoro/loopare ad ogni render (clock, poll rete). Se aggiungi effetti qui,
  **guardali sempre con una key**, altrimenti causi loop di render (schermo bloccato).
- Niente `import`/`export`, niente TypeScript, niente build step. JS classico nel browser.

### Sottosistemi principali (dove guardare):
| Area | Metodi / stato chiave |
|---|---|
| i18n (18 lingue) | tabelle `TR, TR_EXTRA, NTR, ATR, UX, RES, TIMES`; accessor `t()/nt()/ux()/rt()/timeName()`; merge+cache in `langTable()` |
| Mappa base | `tryInitMap()`, `fitArea()`, Leaflet con renderer **SVG** |
| Selezione area / disegno | `startPin/startRect/startDraw`, `onDrawPointerDown/Move/Up`, `_endGesture`, `finishPin/finishRect/finishDraw`, `updateDrawLayer/updatePinPreview/updateRectPreview`, `refreshAreaLayer` |
| Geolocalizzazione | `goToMyLocation()` (default raggio 2 km, pin modificabile) |
| Dati live specie | `maybeRefresh` → `refreshLive`; conteggi per gruppo `maybeGroupCounts` → `fetchGroupCounts` |
| Resilienza rete | `jget()` (timeout+backoff, onora 429/503), `measureNet()` |
| Bottom sheet | `sheetDragStart`, `recalcSheet`, snap points |
| Loading | stato `busy` → scrim a schermo intero che blocca input |
| Scheda specie | `openSpecies`, `fetchTaxonInfo`, `fetchSeason`, `syncObsMap` (mappa osservazioni clusterizzata), evidenze iNat+GBIF |
| Persistenza | `localStorage` (`se_mode/se_accent/se_lang/se_favs/se_skip`) + deep-link hash URL (`syncUrl/restoreUrl`) |

---

## 3. Modello dei conteggi (importante: i numeri DEVONO combaciare)

Fonte autorevole = `species_counts.total_results` (MAI la lunghezza della lista, che è
limitata a 50 righe).
- **Header senza filtro periodo** = specie presenti nell'area (`areaTotal`).
- **Header con filtro periodo/ora** = specie che rispettano il filtro (`observedN`), +
  riga di contesto "presenti nell'area" = `areaTotal` (tutto l'anno).
- **Chip per gruppo** = stessi filtri → **la loro somma = numero grande**. Se aggiungi
  filtri, applicali sia ai chip (`fetchGroupCounts`) sia all'header (`refreshLive`),
  altrimenti i numeri non combaciano (bug storico già corretto).

---

## 4. Storia recente e bug noti (contesto per non ripetere errori)

- **Freeze su tap ravvicinati sulle opzioni area** → mitigato: teardown unico dei
  listener di gesto (`_endGesture`), throttle del drag raggio, mappa sempre ri-abilitata
  quando non si disegna, e lo scrim di loading blocca gli input a raffica. ⚠️ Va ancora
  **verificato su dispositivo con passi di riproduzione precisi**.
- **Punti/area invisibili mentre si disegna** → risolto: tolto `preferCanvas` (il canvas
  non ridisegnava i vettori finché la mappa non si muoveva); i **vertici sono marker DOM**
  (`vtxIcon`, sempre visibili), poligono/cerchio in SVG.
- **Regressione i18n**: le traduzioni erano state spostate in un file `se-i18n.js`
  separato → sul telefono non si caricava (nessun service worker che lo mettesse in cache)
  e l'app andava in **schermo bianco**. **Reinlinate nel file unico.** Lezione:
  per questa PWA condivisibile, **meno file = più affidabile**; non ri-splittare i dati
  necessari al primo render senza un service worker che li precachi.

---

## 5. Primo task consigliato: rete di sicurezza (error boundary)

Oggi un'eccezione in `renderVals()`/render → **pagina bianca** con una barra rossa. Serve
un *error boundary*: se il render lancia, mostrare una card "Qualcosa è andato storto —
Ricarica" **con il testo dell'errore**, invece di distruggere l'app. Trasforma i crash in
situazioni recuperabili e ti dà i log dal dispositivo. (Bonus: log opzionale dell'errore
in `localStorage` per poterlo rileggere.)

Altri task aperti utili:
- **PWA offline vera**: aggiungere un service worker che precacha `index.html`, `support.js`,
  `manifest.webmanifest`, le icone e gli asset Leaflet, con versioning della cache.
- Riprodurre e chiudere definitivamente il freeze con test su dispositivo.
- Valutare vector tiles (vedi §1) prima di qualsiasi uso non personale.

---

## 6. Deploy / esecuzione

Sono file **statici**, nessun build. Per provarli basta servire la cartella `release/`
con un web server statico e aprire `index.html`. Dettagli operativi in `release/DEPLOY.md`.
Per testare come PWA su telefono serve **HTTPS** (es. deploy su Netlify/Vercel/Cloudflare
Pages, o `localhost`).

---

## 7. File in questo pacchetto
- `app/index.html` — l'app completa (template + logica).
- `app/support.js` — runtime del framework (**non modificare**).
- `app/manifest.webmanifest` — manifest PWA.
- `app/DEPLOY.md` — note di deploy e sui limiti dei servizi dati.
- `HOW_TO_CLAUDE_CODE.md` — come far lavorare Claude Code sul tuo repo (test + scrittura).
