# Species Distribution — Deploy & aggiornamento

Guida per mettere l'app online (gratis) e **aggiornarla in pochi click**.
Nessun build, nessun server, nessun database: sono file statici.

---

## 1. Cosa stai pubblicando

L'intera cartella. I file che contano:

```
index.html                    ← l'app (si apre sul dominio nudo)
support.js                    ← runtime che carica (deve stare accanto)
manifest.webmanifest          ← rende installabile come app
deploy/icon-192.png           ← icone
deploy/icon-512.png
deploy/icon-180.png
```

Tienili insieme, stessa struttura di cartelle. Tutto il resto (mappe, font, dati
specie) viene caricato dal vivo da servizi pubblici gratuiti — non lo ospiti tu.

> `index.html` si apre sul dominio nudo (es. `https://tuosito.pages.dev/`), senza
> nome file nell'URL.

---

## 2. ⚠️ Perché il "Direct Upload" ti ha bloccato

Se carichi la cartella con **Direct Upload** (trascinamento su Cloudflare Pages),
NON puoi più modificare i file: per ogni modifica devi ri-caricare tutta la
cartella a mano. È un vicolo cieco per gli aggiornamenti.

**Soluzione: collega Cloudflare Pages (o GitHub Pages) a un repository Git.**
Così aggiorni un file dal browser e il sito si ripubblica da solo.

### Pulizia del vecchio progetto Direct Upload
1. Cloudflare → **Workers & Pages** → apri il progetto.
2. **Settings** → in fondo **Delete project** (conferma digitando il nome).
   - In alternativa, per tenerlo: tab **Deployments** → **…** → **Delete** sui
     deployment vecchi.
3. Poi ricrealo collegato a Git (sezione 3).

---

## 3. Metodo consigliato — repo Git + deploy automatico

### 3a. Crea il repository (una volta sola)
1. Crea un account gratuito su **GitHub** e un **nuovo repository**.
   - Pubblico se userai GitHub Pages; con Cloudflare Pages va bene anche privato.
2. Carica i file **nella root** del repo (drag-and-drop nella UI web di GitHub va
   benissimo): `index.html`, `support.js`, `manifest.webmanifest`, cartella
   `deploy/`, e questo `DEPLOY.md`.
   - Importante: `index.html` deve stare nella **root**, non in una sottocartella.

### 3b. Collega l'hosting (una volta sola) — scegline UNO

**Cloudflare Pages (consigliato)**
1. Cloudflare → **Workers & Pages → Create → Pages → Connect to Git**.
2. Autorizza GitHub e scegli il repo.
3. Impostazioni build:
   - **Framework preset: None**
   - **Build command: (vuoto)**
   - **Build output directory: `/`**
4. **Save and Deploy**. Pubblica su `https://<progetto>.pages.dev`.

**Oppure GitHub Pages (ancora più semplice)**
1. Repo → **Settings → Pages**.
2. "Build and deployment" → Source: **Deploy from a branch** →
   Branch: `main`, cartella: **/ (root)** → **Save**.
3. Dopo ~1 minuto il sito è su `https://<utente>.github.io/<repo>/`.

Entrambi servono su **HTTPS**, richiesto per "installa sulla home" e per le API
di localizzazione.

---

## 4. Aggiornare l'app — in 3 click

Con il repo collegato, per qualsiasi modifica:

1. Vai sul file su **github.com** (es. `index.html`).
2. Clicca l'icona **matita ✏️** (Edit), modifica il testo.
3. **Commit changes**.

→ Cloudflare/GitHub ricostruisce e ripubblica da solo in ~30 secondi.
Stesso URL. Chi ha già installato la PWA riceve la nuova versione alla prossima
apertura — **nessuna re-installazione**.

> Preferisci lavorare da computer? `git clone` del repo, modifichi in locale,
> `git push`: stesso risultato automatico. In questa fase **un solo ambiente**
> (niente dev/prod separati): il repo è l'unica fonte di verità.

---

## 5. Installare sul telefono

L'app è una **PWA**: si installa dal browser, senza app store.

### iPhone / iPad (iOS 16.4+)
1. Apri l'URL in **Safari** (obbligatorio per l'installazione su iOS).
2. Tocca **Condividi** (quadrato con freccia su).
3. Scorri → **Aggiungi a Home** → **Aggiungi**.

### Android (Chrome / Edge / Samsung Internet)
1. Apri l'URL in **Chrome**.
2. Tocca il banner **"Installa app"**, oppure **⋮ menu → Installa app**.
3. Conferma.

---

## 6. iOS e Android? Sì, un solo codice per entrambi

È un'app web standard: gira su iOS (Safari + PWA), Android (Chrome/Edge/Samsung/
Firefox + PWA) e desktop. Niente da mantenere per piattaforma; stesso URL e stessi
file su ogni dispositivo. Su telefono riempie tutto lo schermo; su desktop mostra
la mappa a tutto schermo con i controlli in una colonna centrata.

---

## 7. Costi — cosa resta gratis

- **Hosting**: piano gratuito su Cloudflare Pages / GitHub Pages, ampiamente
  sufficiente.
- **Mappe (tile OpenStreetMap)**: gratis. ⚠️ Il server tile pubblico OSM va bene
  per traffico basso ma la sua policy vieta l'uso pesante in produzione. Se cresce,
  passa a tile vettoriali (Protomaps `.pmtiles` su Cloudflare R2): è una modifica
  di una riga nel setup mappa.
- **Dati specie (API iNaturalist + GBIF)**: gratis, senza chiave. L'app già
  raggruppa, va in timeout e riprova.
- **Confini + ricerca luoghi**: gratis. La ricerca usa **Photon** (geocoder open
  source, endpoint pubblico, nessuna chiave) con cache in-sessione e retry
  automatico silenzioso se il servizio è momentaneamente sotto carico; i confini
  usano OpenStreetMap Nominatim (1 sola richiesta alla selezione). Per uso pesante,
  self-host di Photon/Nominatim — solo se il volume cresce.

---

## 8. Checklist prima di condividere il link

- [ ] Aprendo l'URL compare la schermata "What are you looking for?".
- [ ] La mappa si carica e puoi fare pan/zoom.
- [ ] La ricerca luoghi mostra risultati reali (es. "Roma") e scegliendone uno
      disegna il confine vero.
- [ ] Disegno area e pin funzionano sul telefono.
- [ ] Sul telefono: Aggiungi a Home mostra l'icona Species e apre a schermo intero.
