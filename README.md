# podcast-audiogram-automation

Orchestratore **multi-podcast** per la generazione e pubblicazione automatica di
audiogram, su runner **GitHub-hosted** (`ubuntu-latest`, nessun runner self-hosted).

Questo repo è il *control plane*: sa quali podcast esistono, quali feed, quali
social, quando girare e dove archiviare. Il lavoro pesante lo fanno due tool
generici e separati, richiamati dai workflow:

- [`podcast-audiogram-generator`](https://github.com/valeriogalano/podcast-audiogram-generator) — feed RSS → audiogram `.mp4`
- [`podcast-audiogram-publisher`](https://github.com/valeriogalano/podcast-audiogram-publisher) — `.mp4` → social

---

## Architettura

Tre repo con ruoli distinti (i tool restano separati e riutilizzabili):

| Repo | Ruolo | Dipendenze |
|---|---|---|
| `podcast-audiogram-generator` | Tool: feed RSS → audiogram `.mp4` | moviepy, ffmpeg |
| `podcast-audiogram-publisher` | Tool: `.mp4` → social | google-api, telethon, mastodon |
| **questo repo** | Orchestratore: config per podcast, workflow, stato, archivio | — |

I tool sono generici (tutto in config, accettano `--config`/`--state-file`) e non
vanno modificati per il multi-podcast: è tutto orchestrazione qui.

### Struttura

```
podcasts/
  pensieriincodice/
    generate.yaml      # config generator (template con ${VAR} risolti a runtime)
    publish.yaml       # config publisher: piattaforme abilitate per questo podcast
    published.json     # stato pubblicazioni (aggiornato dai run)
.github/workflows/
  _pipeline.yml        # pipeline riutilizzabile (workflow_call)
  pensieriincodice.yml # caller: schedule + environment
docs/PLAN.md           # piano della migrazione
```

---

## Il flusso della pipeline

I runner GitHub-hosted sono **effimeri**: il filesystem sparisce a fine job. La
pipeline ricrea lo stato persistente (che sul vecchio Mac stava su disco) usando
le **GitHub Releases** come archivio e committando lo stato nel repo. Gli step, in
ordine:

1. **restore** — scarica dalle Release recenti gli asset (i file dell'archivio) e
   **ricostruisce** `output/epNNN/sbM/` dai nomi file. Serve a far ripartire il
   runner dallo stato precedente, così il generator è *idempotente*.
2. **reset** *(solo con `force` + `episode`)* — azzera in `published.json` le voci
   dell'episodio indicato, per poterlo rigenerare/ripubblicare da capo.
3. **generate** — il generator produce gli audiogram. Senza `--force` **salta**
   ciò che è già presente (dopo il restore), quindi genera solo i soundbite nuovi,
   fino a `generate_limit` per run.
4. **archive** — carica i file prodotti come **asset singoli** (non uno zip) della
   Release `<podcast>-epNNN`. È l'archivio scaricabile file per file, utile anche
   per la pubblicazione manuale. Il `restore` li ricostruisce dai nomi.
5. **publish** — il publisher pubblica il prossimo soundbite non ancora pubblicato
   (`publish_limit` per run) sulle piattaforme con `enabled: true`.
6. **commit stato** — `published.json` viene committato di ritorno nel repo
   (saltato in `dry_run`). Avviene in `full`/`publish` e **anche in `generate`
   quando lo step *reset* è girato**: così un force-regen azzera lo stato in modo
   persistente e la successiva pubblicazione ripubblica i soundbite rigenerati.
   I run dello stesso podcast sono accodati, non cancellati, e prima del push
   dello stato il workflow fa `git pull --rebase --autostash` sul branch corrente:
   un avanzamento di `main` durante il job non rende il push non-fast-forward.

### Comportamento incrementale

`generate_limit` (default **3**) significa "genera al massimo N soundbite **non
ancora generati** per run" — il vecchio approccio incrementale del Mac, preservato
grazie a `restore` + skip-dei-già-fatti. Su cadenza settimanale l'archivio di un
episodio si completa in più run (es. 5 soundbite con limit 3 → 2 run). Il publisher
a sua volta pubblica `publish_limit` (default **1**) soundbite per run.

### Aggiungere un soundbite dopo che la Release esiste

L'identità di un soundbite è **posizionale** (`sb1`, `sb2`, …), non basata sul
contenuto:

- **In coda** (diventa `sb4`/`sb5`): pulito. Il `restore` riporta i precedenti, il
  generator li salta e genera solo il nuovo; il publisher lo vede come prossimo da
  pubblicare. Nessuna azione speciale.
- **Inserito a metà / riordinato** (gli indici slittano): archivio e
  `published.json` diventano incoerenti, perché file e stato sono indicizzati per
  posizione. Per rimediare, rigenerare l'episodio con **`force` + `episode`**: lo
  step *reset* azzera lo stato dell'episodio e `--force` riscrive i video.

> Nota: con la config attuale (`episode: last`, `restore_releases: 1`) i run
> automatici toccano solo l'episodio più recente. Per intervenire su un episodio
> più vecchio, usa un avvio manuale con `episode` esplicito (vedi sotto).

---

## Modalità e input

### Modalità (`mode`)

| Modalità | Step eseguiti | Uso tipico |
|---|---|---|
| `full` | restore → [reset] → generate → archive → publish → commit | Tutto in un colpo (avvio manuale) |
| `generate` | restore → [reset] → generate → archive → [commit stato se reset] | Solo produzione + archivio (schedule del martedì) |
| `publish` | restore → publish → commit | Solo pubblicazione (schedule del giovedì) |

`[reset]` avviene solo se `force=true` con un `episode` specifico. Quando il reset
gira, lo stato azzerato viene committato in **qualunque** modalità (anche
`generate`), così il force-regen persiste per la pubblicazione successiva.

### Input dell'avvio manuale (`workflow_dispatch`)

| Input | Default | Descrizione |
|---|---|---|
| `mode` | `full` | `full` \| `generate` \| `publish` |
| `dry_run` | `false` | Publisher in `--dry-run`: nessun upload, nessun commit dello stato |
| `episode` | *(vuoto)* | Episodio da processare (vuoto = da config: `last`). Es. `150` |
| `force` | `false` | Rigenera con `--force` **e** azzera lo stato dell'episodio indicato |

### Input avanzati della pipeline riutilizzabile (`_pipeline.yml`)

Non esposti nel caller, hanno default sensati; si cambiano modificando il caller:

| Input | Default | Descrizione |
|---|---|---|
| `generate_limit` | `3` | Max soundbite generati per episodio per run |
| `publish_limit` | `1` | Max soundbite pubblicati per run |
| `restore_releases` | `1` | Quante Release recenti ripristinare prima del run |
| `generator_repo` / `generator_ref` | `…/podcast-audiogram-generator` / `main` | Sorgente e versione del generator |
| `publisher_repo` / `publisher_ref` | `…/podcast-audiogram-publisher` / `main` | Sorgente e versione del publisher |

---

## Esecuzione

### Schedule (automatico)

| Giorno | Modalità | Effetto |
|---|---|---|
| Martedì 10:00 UTC | `generate` | Genera e archivia i nuovi soundbite dell'ultimo episodio |
| Giovedì 10:00 UTC | `publish` | Pubblica il prossimo soundbite non pubblicato |

### Manuale (`gh` o tab Actions → *Run workflow*)

```bash
# Anteprima completa senza pubblicare né toccare lo stato
gh workflow run pensieriincodice.yml -f mode=full -f dry_run=true

# Rigenera da capo l'episodio 150 e riazzera il suo stato di pubblicazione
gh workflow run pensieriincodice.yml -f mode=generate -f episode=150 -f force=true

# Solo pubblicazione del prossimo soundbite in coda
gh workflow run pensieriincodice.yml -f mode=publish
```

---

## Archivio (GitHub Releases)

- Una Release per episodio, tag `<podcast>-epNNN` (es. `pensieriincodice-ep150`).
- Asset = **file singoli** (`ep150_sb1_nosubs_vertical.mp4`, caption, ecc.),
  scaricabili singolarmente per la pubblicazione manuale.
- Gli asset stanno fuori dal repo: la history git resta pulita.
- GitHub aggiunge in automatico gli asset "Source code (zip/tar.gz)": qui sono
  rumore inevitabile (il tag è solo un contenitore di video), si ignorano.

---

## Secret (per podcast → GitHub Environment)

I secret stanno in un **Environment GitHub** omonimo del podcast (es.
`pensieriincodice`), mai nel repo. Per un podcast con YouTube + Telegram:

| Secret | Contenuto |
|---|---|
| `YOUTUBE_CLIENT_SECRETS` | contenuto JSON di `youtube_client_secrets.json` |
| `YOUTUBE_TOKEN` | contenuto JSON di `youtube_token.json` |
| `TELEGRAM_API_ID` | api_id Telegram |
| `TELEGRAM_API_HASH` | api_hash Telegram |
| `TELEGRAM_SESSION_B64` | `base64` di `secrets/telegram.session` |

> Il reusable richiede `contents: write` (commit stato + Release): il caller lo
> concede con un blocco `permissions:`, perché il default del `GITHUB_TOKEN` del
> repo è read-only.

---

## Aggiungere un podcast

1. Crea `podcasts/<nome>/` con `generate.yaml`, `publish.yaml`, `published.json` (`{}`).
2. Crea l'Environment GitHub `<nome>` e caricaci i secret delle sue piattaforme.
3. Duplica un caller (es. `pensieriincodice.yml`) come `<nome>.yml`, impostando
   `podcast: <nome>`, `environment: <nome>` e lo schedule desiderato.

Solo le piattaforme con `enabled: true` in `publish.yaml` vengono pubblicate: un
podcast può avere un sottoinsieme dei social.

## Podcast configurati

| Podcast | Feed | Piattaforme | Schedule |
|---|---|---|---|
| Pensieri in codice | pensieriincodice.it | YouTube, Telegram | Mar 10:00 UTC (genera) · Gio 10:00 UTC (pubblica) |
