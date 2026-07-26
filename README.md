# Monitor Concerti Accessibili

Pagina web che controlla automaticamente **6 volte al giorno** i siti dove
vengono pubblicati i concerti con procedura di accesso dedicata alle persone
con disabilità, e segnala in cima alla pagina i **nuovi eventi** appena
comparsi. Notifica opzionale su **Telegram**.

Tutto gira su GitHub: hosting gratuito (GitHub Pages) e automazione gratuita
(GitHub Actions). Non serve un server e non serve tenere il PC acceso.

---

## Cosa fa

- Scansiona i siti elencati in `sites.yaml` alle **07, 09, 13, 15, 19, 21**
  (ora italiana — 2 la mattina, 2 il pomeriggio, 2 la sera)
- Riconosce gli eventi mai visti prima e li mette **in evidenza in cima alla
  pagina per 2 giorni**, con il badge verde `NUOVO`
- Mantiene sotto l'**elenco completo**, filtrabile e ordinabile per
  promoter, città e data cronologica
- Per ogni evento mostra **data, luogo, città, promoter e come fare richiesta**
- Manda un messaggio **Telegram** appena trova qualcosa di nuovo

## Siti monitorati di partenza

| Sito | Perché |
|---|---|
| **mani-amiche.it** (agenda + home) | L'hub centrale: gestisce gli accrediti per MC2 Live, Vertigo e molti altri |
| **mc2live.it** | Procedure accesso disabilità |
| **vertigo.co.it** (FAQ + tour) | Modalità accesso disabili |
| **puntoeacapo.uno** | Tabella eventi con email dedicata per artista |
| **friendsandpartners.it** | Modulo prenotazione disabilità |
| **dalessandroegalli.com** | Promoter eventi |
| **ticketone.it** | Sezione dedicata |

Aggiungerne altri richiede solo di incollare un blocco in `sites.yaml`.

---

## Installazione (circa 10 minuti)

### 1. Crea il repository

Su GitHub: **New repository** → nome a piacere (es. `concerti`) →
**Public** (necessario per avere GitHub Pages gratis) → **Create**.

Poi carica tutti i file di questa cartella: sulla pagina del repo vuoto usa
**uploading an existing file**, trascina tutto e fai commit.

> Attenzione: trascinando la cartella, GitHub mantiene la struttura. Verifica
> che esista `.github/workflows/monitor.yml`. Se la cartella `.github` non
> compare (alcuni browser saltano le cartelle che iniziano con punto), creala
> a mano con **Add file → Create new file** scrivendo il percorso
> `.github/workflows/monitor.yml` e incollandoci dentro il contenuto.

### 2. Attiva GitHub Pages

**Settings → Pages → Source: GitHub Actions**.

### 3. Dai i permessi di scrittura alle Actions

**Settings → Actions → General** → in fondo, *Workflow permissions* →
seleziona **Read and write permissions** → **Save**.

Senza questo passaggio il workflow non riesce a salvare i dati.

### 4. Telegram (opzionale ma consigliato)

1. Su Telegram scrivi a **@BotFather** → `/newbot` → scegli un nome →
   ricevi un **token** tipo `123456789:AAF...`
2. Scrivi un messaggio qualsiasi al tuo nuovo bot (serve per sbloccarlo)
3. Apri nel browser
   `https://api.telegram.org/bot<IL_TUO_TOKEN>/getUpdates`
   e copia il numero in `"chat":{"id":123456789`
4. Sul repo: **Settings → Secrets and variables → Actions → New repository secret**
   - `TELEGRAM_TOKEN` = il token di BotFather
   - `TELEGRAM_CHAT_ID` = il numero del punto 3
5. Nella scheda **Variables** della stessa pagina aggiungi
   - `PAGES_URL` = `https://TUONOME.github.io/NOMEREPO/`
   (serve solo per mettere il link nel messaggio Telegram)

### 5. Primo avvio

Vai su **Actions → Monitor Concerti → Run workflow**.
Dopo un paio di minuti la pagina è online su
`https://TUONOME.github.io/NOMEREPO/`

---

## Aggiungere un sito

Apri `sites.yaml` direttamente su GitHub (matita in alto a destra) e aggiungi:

```yaml
  - name: "Nome che vedrai in pagina"
    promoter: "Nome Promoter"
    url: "https://esempio.it/eventi"
    parser: "generic"
    enabled: true
    richiesta: >-
      Spiegazione di come si fa richiesta biglietti.
```

Fai commit: il workflow riparte da solo e il sito è incluso dal giro dopo.

Per **disattivare** un sito senza cancellarlo, metti `enabled: false`.

### I due parser

- **`generic`** — funziona ovunque senza configurazione. Cerca nel testo
  frammenti che contengono una data riconoscibile insieme a una parola chiave
  o a una città italiana. È il default: usalo per qualsiasi sito nuovo.
- **`table`** — più preciso, per siti con una tabella regolare. Richiede di
  indicare la posizione delle colonne (`columns:`, si conta da 0). Se la
  tabella cambia struttura, il codice ricade automaticamente su `generic`.

---

## Cambiare gli orari

In `.github/workflows/monitor.yml`, nella sezione `schedule`.
**Gli orari cron sono in UTC**: l'Italia è UTC+2 d'estate e UTC+1 d'inverno.
Quindi `0 5 * * *` = 07:00 italiane d'estate, 06:00 d'inverno.

> Nota: i cron di GitHub Actions su repository gratuiti possono partire con
> qualche minuto di ritardo quando i server sono carichi. Per non perdere le
> aperture prenotazioni, meglio tenere 6 controlli al giorno come impostato
> che pretendere la puntualità al minuto.

---

## Come funziona internamente

```
sites.yaml                 configurazione: quali siti, come leggerli
scripts/scraper.py         scarica, estrae eventi, confronta col passato
scripts/build_page.py      genera docs/index.html
scripts/notify.py          manda i nuovi eventi su Telegram
scripts/test_local.py      test offline dei parser
data/events.json           archivio completo (con first_seen di ogni evento)
data/new.json              solo i nuovi dell'ultimo giro
docs/index.html            la pagina pubblicata
```

Ogni evento ha un'impronta calcolata da sito + titolo + data + città. Se
l'impronta non esisteva, è un evento nuovo: viene registrato il `first_seen`
e parte la notifica. Il `first_seen` non cambia più, per questo la sezione
"Novità" sa quali eventi sono comparsi negli ultimi 2 giorni.

Se un sito è temporaneamente irraggiungibile, gli eventi futuri già noti
restano in pagina invece di sparire.

Per cambiare i giorni di permanenza in evidenza, modifica
`giorni_in_evidenza` in `sites.yaml`.

## Test in locale (facoltativo)

```bash
pip install -r requirements.txt
python scripts/test_local.py          # test offline dei parser
python scripts/scraper.py --dry-run   # prova sui siti veri, non salva niente
```

---

## Avvertenza

Le procedure di accredito cambiano spesso e senza preavviso, e ogni promoter
ha regole sue. **Verifica sempre sul sito ufficiale** prima di inviare una
richiesta: questa pagina serve ad accorgersi in fretta che qualcosa è uscito,
non a sostituire il canale ufficiale.
