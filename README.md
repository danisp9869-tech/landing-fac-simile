# 🍽️ Template Landing Prenotazioni — Food Media Lab

> ## 👉 Come usare questo template
> Questo repository è un **modello riutilizzabile**. Per creare la landing di un
> nuovo cliente **non modificare questo repo**: usa il pulsante verde
> **"Use this template" → "Create a new repository"** in alto.
> Otterrai una copia indipendente; lì apri `index.html`, modifica solo il blocco
> **`CONFIG`** con i dati del cliente e il sito si pubblica da solo su GitHub Pages.
> Dettagli nella sezione **§1** qui sotto.

Template **single-file** (`index.html`) di landing page per ristoranti, il cui unico
scopo è **far prenotare un tavolo**. Nessuna dipendenza, nessun build: si apre con un
doppio click. Per ogni nuovo cliente si modifica **solo l'oggetto `CONFIG`** in cima al
file `index.html`.

Sezioni della pagina: **Hero** (logo + titolo + sottotitolo + CTA) → **Form di
prenotazione intelligente** (con gestione capienza reale) → **Footer** (contatti, orari,
mappa, social).

---

## ✅ Checklist rapida per ogni nuovo cliente

1. **Crea il repo** dal template ("Use this template").
2. **Attiva GitHub Pages** (passaggio manuale, una volta): repo → **Settings → Pages →
   Build and deployment → Source: "GitHub Actions"**. Senza questo la pubblicazione fallisce
   con *"Create Pages site failed"*.
3. **Personalizza il `CONFIG`** in `index.html` con i dati del cliente (e il blocco `DATI`
   in `privacy.html`). A ogni salvataggio su `main` il sito si ripubblica da solo.
4. **Google Sheet**: crea il foglio + la Web App (vedi §3) e incolla l'URL in
   `googleSheetsWebAppUrl`. ⚠️ Accesso Web App = **"Chiunque"**, e **dopo ogni modifica dello
   script ripubblica con "Nuova versione"**.
5. **Svuota i dati demo**: `occupatiDemo: {}` e `ancoraDemo: false`.
6. **Prova**: fai una prenotazione di test e verifica che arrivi nel foglio, poi cancella la
   riga di test.

> I due passaggi che si dimenticano più spesso: **(2)** attivare Pages come "GitHub Actions"
> e **(4)** ripubblicare lo script con "Nuova versione" dopo ogni modifica.

---

## 1. Creare una landing per un nuovo cliente

Apri `index.html`, trova il blocco `const CONFIG = { … }` (è tutto commentato in
italiano) e modifica solo questi campi:

| Blocco `CONFIG` | Cosa contiene |
|---|---|
| `brand` | Nome ristorante, tagline, `logo` (path immagine) o `iniziali` (fallback). |
| `colori` | Palette. Default nero/bianco/oro FML. `ctaBg` / `ctaText` = colore del bottone. |
| `hero` | `headline` (usa `<em>parola</em>` per l'oro corsivo), `sottotitolo`, testo CTA. |
| `contatti` | Indirizzo, telefono, email, link mappe, social, righe orari del footer. |
| `prenotazioni` | **Capienza, fasce orarie, giorni di chiusura** (vedi §2). |
| `integrazione` | URL del Google Sheet del cliente (vedi §3). |
| `testi` | Etichette fisse dell'interfaccia (di norma non serve toccarle). |

> Il logo: metti il file immagine nella stessa cartella e imposta `brand.logo: "logo.png"`.
> Se lo lasci vuoto (`""`), viene mostrato il cerchio con le iniziali di `brand.iniziali`.

---

## 2. Capienza e fasce orarie

Tutto dentro `CONFIG.prenotazioni`:

```js
capienzaCoperti: 50,   // coperti totali disponibili PER OGNI fascia oraria
fasceOrarie: ["19:00","19:30","20:00","20:30","21:00","21:30","22:00"],
giorniChiusura: [1],   // 0=Dom 1=Lun … 6=Sab  → qui: chiuso il lunedì ([] = mai)
maxPersonePrenotazione: 20,
sogliaUltimiPosti: 8,  // sotto N coperti liberi → badge "Ultimi posti"
giorniPrenotabili: 60, // finestra di prenotazione (giorni da oggi)
```

### Come funziona la disponibilità (il "grigetto")
- Ogni fascia oraria ha un tetto pari a `capienzaCoperti`.
- Il sistema somma i coperti già prenotati per **data + fascia**.
- Se una fascia è piena → diventa **grigia, non cliccabile**, badge **"Completo"**.
- Considera anche le **persone della prenotazione in corso**: se restano 3 posti e
  l'utente ha scelto 4 persone, quella fascia diventa grigia ("Solo 3 posti"). La
  disponibilità viene **ricalcolata ogni volta che si cambia il numero di persone**.
- Quando restano pochi posti (≤ `sogliaUltimiPosti`) → badge **"Ultimi posti"**.

### Dati di esempio per la demo
`prenotazioni.occupatiDemo` contiene coperti finti già occupati, così vedi subito la
logica funzionare. Le chiavi `"GIORNO+1"`, `"GIORNO+2"` vengono convertite in date reali
(oggi + N) grazie a `ancoraDemo: true`. **In produzione svuota `occupatiDemo`** (`{}`):
i conteggi arriveranno dallo Sheet / dal DB.

Le prenotazioni fatte dall'utente vengono salvate in **localStorage**, così ricaricando la
pagina i conteggi restano coerenti e le fasce piene restano grigie.

---

## 3. Collegare il Google Sheet (uno diverso per ogni ristorante)

Ogni landing invia le prenotazioni a uno **Sheet dedicato** tramite una **Web App di
Google Apps Script**. Serve solo incollare un URL in `CONFIG.integrazione.googleSheetsWebAppUrl`.

### Passi (una volta per cliente)
1. Crea un nuovo Google Sheet per il ristorante. Prima riga (intestazioni):
   `Data | Ora | Persone | Nome | Telefono | Email | Richieste | Privacy | Creata`
2. Prendi l'**ID del foglio** dall'URL: è la parte lunga **tra `/d/` e `/edit`**.
3. Menu **Estensioni → Apps Script**, cancella tutto e incolla (inserendo il tuo ID):

   ```js
   const SHEET_ID = "INCOLLA_QUI_L_ID_DEL_FOGLIO";   // ⚠️ SOLO l'ID, senza /d/ e senza /edit

   function doPost(e) {
     const sh = SpreadsheetApp.openById(SHEET_ID).getSheets()[0];
     const d = JSON.parse(e.postData.contents);
     sh.appendRow([d.data, d.ora, d.persone, d.nome, d.telefono, d.email, d.richieste, d.privacy, d.creata]);
     return ContentService.createTextOutput(JSON.stringify({ ok: true }))
                          .setMimeType(ContentService.MimeType.JSON);
   }

   // (Opzionale) legge i coperti occupati per far funzionare il grigetto "dal vivo".
   // Attiva anche CONFIG.integrazione.leggiOccupatiDaSheets = true.
   function doGet(e) {
     const rows = SpreadsheetApp.openById(SHEET_ID).getSheets()[0].getDataRange().getValues();
     const out = {};
     for (let i = 1; i < rows.length; i++) {
       const [data, ora, persone] = rows[i];
       if (!data) continue;
       const k = Utilities.formatDate(new Date(data), 'GMT', 'yyyy-MM-dd');
       out[k] = out[k] || {};
       out[k][ora] = (out[k][ora] || 0) + Number(persone || 0);
     }
     return ContentService.createTextOutput(JSON.stringify(out))
                          .setMimeType(ContentService.MimeType.JSON);
   }
   ```
   > ℹ️ Si usa **`openById(SHEET_ID)`** e **non** `getActiveSpreadsheet()`: così funziona anche
   > quando lo script è un "progetto separato" e non un progetto legato al foglio (caso comune).
4. **Distribuisci → Nuova distribuzione → Tipo: App web**. Alla voce **"Chi ha accesso"**
   scegli **"Chiunque"** — ⚠️ *non* "Chiunque con un account Google", è la causa n°1 dei dati
   che non arrivano. Autorizza fino a **"Consenti"**. Copia l'URL `…/exec` e incollalo in:
   ```js
   integrazione: { googleSheetsWebAppUrl: "https://script.google.com/macros/s/XXXX/exec" }
   ```
5. **⚠️ IMPORTANTE — dopo OGNI modifica al codice dello script**, ripubblica, altrimenti l'URL
   continua a usare la versione vecchia (i dati non si aggiornano):
   **Distribuisci → Gestisci distribuzioni → ✏️ (matita) → Versione: "Nuova versione" → Distribuisci**.
   L'URL **resta identico**, non serve ritoccare la landing.
6. (Opzionale) per leggere le disponibilità reali dallo Sheet all'avvio:
   `integrazione.leggiOccupatiDaSheets: true`.

> 🧪 **Test rapido se i dati non arrivano:** nell'editor Apps Script aggiungi
> `function test(){ SpreadsheetApp.openById(SHEET_ID).getSheets()[0].appendRow(["TEST", new Date()]); }`,
> selezionala dal menu a tendina in alto e premi **▶ Esegui**. Se la riga compare, lo script
> scrive bene → controlla accesso "Chiunque" e ripubblica con "Nuova versione". Se non compare,
> è l'ID del foglio o l'autorizzazione.

Se lasci `googleSheetsWebAppUrl: ""`, la landing funziona in **solo-locale** (demo): salva
in localStorage e mostra la conferma, senza invii esterni.

---

## 4. Passare a un DB reale (Supabase) in futuro

La logica di disponibilità è **isolata in funzioni pure e documentate** dentro
`index.html`. Per migrare basta cambiare una funzione, senza toccare la UI:

| Funzione | Ruolo | Cosa cambiare per Supabase |
|---|---|---|
| `getPostiOccupati(data, ora)` | Coperti già occupati per data+fascia | Sostituire con `SELECT COALESCE(SUM(coperti),0) FROM bookings WHERE data=? AND ora=?`. |
| `getDisponibilita(data, ora)` | `capienza − occupati` | Nessuna modifica. |
| `isFasciaDisponibile(data, ora, persone)` | Fascia selezionabile? | Nessuna modifica. |
| `salvaPrenotazione(prenotazione)` | Persiste la prenotazione | Sostituire l'invio con un `INSERT INTO bookings (…)`. |

Nel codice trovi i commenti che segnano **esattamente** i punti da sostituire (cerca
`IN PRODUZIONE` / `QUANDO PASSERAI A SUPABASE`). La tabella suggerita:

```sql
create table bookings (
  id bigint generated always as identity primary key,
  data date not null,
  ora  text not null,        -- "HH:MM"
  coperti int not null,
  nome text, telefono text, richieste text,
  creata timestamptz default now()
);
```

---

## Requisiti coperti
- ✅ Single file HTML autonomo, zero build, apribile con doppio click.
- ✅ Responsive mobile-first, accessibile (label, focus states, aria).
- ✅ Validazione: data non passata, giorni di chiusura e date oltre finestra non
  selezionabili, telefono valido, persone > 0.
- ✅ Disponibilità per capienza + persone, con stati "Completo" / "Ultimi posti".
- ✅ Persistenza localStorage + gancio Google Sheets per-cliente.
