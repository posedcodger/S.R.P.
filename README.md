# S.R.P. Portal — guida setup

Questo progetto è pronto per essere pubblicato su GitHub Pages con
storage reale su Firebase (Firestore). Segui i passaggi nell'ordine:
prima Firebase, poi GitHub — ti serve la config di Firebase prima di
pubblicare il sito.

## 1. Crea il progetto Firebase (5 min)

1. Vai su https://console.firebase.google.com e accedi con un account Google.
2. "Aggiungi progetto" → nome a piacere (es. `srp-portal`) → continua fino alla creazione.
3. Nella home del progetto, clicca l'icona `</>` ("Aggiungi Firebase alla tua app web").
4. Dai un nome all'app (es. `srp-web`) → **non serve** spuntare Firebase Hosting → "Registra app".
5. Firebase ti mostra un blocco `firebaseConfig = { ... }`: copia tutti i valori.
6. Apri il file `firebase-config.js` in questo progetto e incolla i valori al posto dei placeholder `INCOLLA_QUI_...`.
7. Nel menu laterale di Firebase: **Build → Firestore Database → Crea database**.
   - Scegli una region vicina (es. `eur3 - europe-west`).
   - Modalità: **test mode** per ora (va bene per i primi test; le regole
     definitive sono più sotto, in "Sicurezza").
8. Vai su **Build → Authentication → Get started** → tab "Sign-in method" →
   attiva il provider **Anonymous**. Serve perché il sito autentica ogni
   dispositivo in modo anonimo prima di permettergli di leggere/scrivere.

## 2. Crea il repository GitHub (3 min)

1. Vai su https://github.com/new
2. Nome repo: es. `srp-portal` → Pubblico o privato (se privato, GitHub Pages richiede
   un piano a pagamento per pubblicarlo — per test consiglio **pubblico**).
3. Non aggiungere README/gitignore automatici (li hai già qui).
4. Crea il repository.
5. Carica questi 3 file nel repo (dalla pagina del repo → "Add file" → "Upload files"):
   - `index.html`
   - `firebase-config.js` (già con i tuoi valori compilati)
   - `logo-placeholder.svg` (sostituiscilo con il tuo logo originale, stesso nome file,
     oppure carica il tuo file e aggiorna i riferimenti `src="..."` dentro `index.html`)
6. Commit.

## 3. Attiva GitHub Pages (2 min)

1. Nel repo: **Settings → Pages**.
2. In "Build and deployment" → Source: **Deploy from a branch**.
3. Branch: `main` (o `master`), cartella: `/ (root)` → Save.
4. Dopo 1-2 minuti il sito sarà live su:
   `https://TUO-USERNAME.github.io/srp-portal/`

## 4. Genera i link per ogni card NFC

Il sito legge il livello dall'URL, quindi ogni card va programmata con un link tipo:

```
https://TUO-USERNAME.github.io/srp-portal/?id=SRP-EHR0C7OXFXM3IJ98X429E6AR
```

Gli ID di test attuali sono nel file `index.html`, oggetto `OPERATORS`:
- `SRP-S46X7MYCR0NQWF9XXTSCU285` → L1, Marco Bianchi
- `SRP-IQAT24HWZCNUYL695SN1N7KO` → L2, Elena Ferrari
- `SRP-PQC299E55QJYMKMKQV6WKDGY` → L3, Luca Greco
- `SRP-C4YUZXJ4NVRDMDCCF0OC12WE` → L4, Sofia Romano
- `SRP-EHR0C7OXFXM3IJ98X429E6AR` → L5, Riccardo Dorigo (la tua card)

Per scrivere il link sulla card NFC, usa un'app come **NFC Tools**
(Android/iOS): "Scrivi" → aggiungi record → tipo "URL/URI" → incolla il link.

## 5. Modificare il sito da telefono

Puoi editare `index.html` direttamente dall'app GitHub per iOS/Android, oppure
da `github.dev` (premi `.` sulla tastiera mentre sei sulla pagina del repo da
desktop per aprire un VS Code nel browser). Ogni push aggiorna il sito live
in 1-2 minuti.

## Sicurezza — cosa è già impostato

Rispetto alla prima versione, ora il sito ha:

- **Autenticazione anonima Firebase**: ogni dispositivo che apre il sito riceve
  un'identità univoca (`authorUid`) generata da Firebase, non solo un nome
  dichiarato dal client. Questo impedisce la scrittura anonima pura e associa
  ogni messaggio/file/voce a un dispositivo reale.
- **ID card non prevedibili**: gli ID non sono più `SRP-0A1-01AA` ma stringhe
  lunghe e casuali (es. `SRP-EHR0C7OXFXM3IJ98X429E6AR`). Indovinare l'ID di un
  livello superiore è computazionalmente impraticabile. L'ID stampato sul
  badge fisico può restare come riferimento visivo/estetico — quello che
  conta per l'accesso è il link scritto nel chip NFC.
- **Dati strutturati in collection reali** (non più un unico blob JSON), così
  le regole Firestore qui sotto possono validare ogni scrittura.

Prima di condividere il sito, imposta queste regole in
**Firestore Database → Regole**:

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {

    match /repos/{repoId} {
      allow read: if request.auth != null;
      allow create: if request.auth != null
        && request.resource.data.name is string
        && request.resource.data.name.size() < 100;
      allow update, delete: if false;
    }

    match /files/{fileId} {
      allow read: if request.auth != null;
      allow create: if request.auth != null
        && request.resource.data.name is string
        && request.resource.data.name.size() < 200
        && request.resource.data.repoId is string;
      allow update, delete: if false;
    }

    match /chats/{chatKey}/messages/{msgId} {
      allow read: if request.auth != null;
      allow create: if request.auth != null
        && request.resource.data.text is string
        && request.resource.data.text.size() > 0
        && request.resource.data.text.size() < 500;
      allow update, delete: if false;
    }

    match /private_entries/{entryId} {
      allow read: if request.auth != null;
      allow create: if request.auth != null
        && request.resource.data.title.size() < 100
        && request.resource.data.body.size() < 1000;
      allow update, delete: if false;
    }

    match /private_files/{fileId} {
      allow read: if request.auth != null;
      allow create: if request.auth != null
        && request.resource.data.name is string
        && request.resource.data.name.size() < 200;
      allow update, delete: if false;
    }

    match /operator_settings/{cardId} {
      allow read: if request.auth != null;
      allow write: if request.auth != null;
    }
  }
}
```

Cosa fanno queste regole:
- Solo utenti autenticati (anche in forma anonima) possono leggere/scrivere.
- I messaggi, le voci private e i file, una volta creati, **non possono
  essere modificati né cancellati** da nessuno tramite l'app.
- Vengono validati tipo e lunghezza dei campi principali.
- `operator_settings` (la password della sezione Private) è scrivibile da
  chiunque sia autenticato — chi conosce l'ID esatto di un'altra card
  potrebbe in teoria sovrascriverne la password. Con ID lunghi e casuali
  (fatto sopra) il rischio pratico è basso, ma è una scelta di compromesso:
  una vera protezione richiederebbe Cloud Functions per validare chi può
  scrivere su quale documento.

**Limite onesto**: queste regole verificano *l'autenticazione*, non *quale
livello* dichiara di avere il client — perché il livello arriva dall'URL
`?id=...`, che è informazione lato client per definizione. Con card NFC
fisiche uniche e ID lunghi/casuali (fatto sopra), il rischio pratico è già
molto basso. Per una separazione dei livelli **verificata anche lato
server** (impossibile da bypassare in alcun modo, nemmeno conoscendo un
altro ID) servirebbe una Cloud Function che valida l'ID e rilascia un
custom auth token per quel livello — è un passo in più, fattibile, dimmi se
vuoi che te lo imposti.

**Nota sul primo avvio**: la query sui file (`repo == X` + ordina per data)
richiede un indice composito. Al primo caricamento della sezione Files,
Firebase potrebbe mostrare nella console del browser un errore con un link
diretto "crea indice qui" — clicca il link, Firebase lo crea in automatico
in circa 1 minuto, poi ricarica la pagina.

## Generare nuovi ID per card reali

Gli ID attuali in `OPERATORS` (dentro `index.html`) sono di esempio. Per
generare i tuoi, usa questo comando (richiede Python):

```
python3 -c "import secrets,string; print('SRP-' + ''.join(secrets.choice(string.ascii_uppercase+string.digits) for _ in range(24)))"
```

Aggiorna l'oggetto `OPERATORS` con i nuovi ID, poi scrivi il link
corrispondente (`https://tuosito/?id=SRP-...`) su ogni chip NFC.
