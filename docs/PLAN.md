# Piano: da runner self-hosted a orchestratore multi-podcast su GitHub-hosted

> Documento di piano. Lo stato "fatto / da fare" è indicato per fase.
> Repo di destinazione (nuovo nome): **`podcast-audiogram-automation`** (pubblico).

## Obiettivo

Eliminare il runner self-hosted sul Mac ("MarkXVI") — instabile e macchinoso — e
far girare tutta l'automazione audiogram su **runner GitHub-hosted** (`ubuntu-latest`),
gratuiti e illimitati perché il repo è **pubblico**.

Il repo evolve da semplice contenitore di workflow a **orchestratore multi-podcast**:
sa quali podcast esistono, quali feed, quali social, quando girare, dove archiviare.

## Architettura

Tre repo con ruoli distinti (i due tool restano **separati e generici**):

| Repo | Ruolo | Dipendenze |
|---|---|---|
| `podcast-audiogram-generator` | Tool: feed RSS → audiogram `.mp4` | moviepy, ffmpeg |
| `podcast-audiogram-publisher` | Tool: `.mp4` → social | google-api, telethon, mastodon |
| **questo repo** (`podcast-audiogram-automation`) | Orchestratore: config per podcast, workflow, stato, archivio | — |

I tool sono già pubblici, generici (tutto in config, accettano `--config`/`--state-file`)
e non vanno modificati per il multi-podcast: è tutto lavoro di orchestrazione qui.

### Layout del repo

```
podcasts/
  pensieriincodice/
    generate.yaml      # config generator (template: contiene ${VAR} risolti a runtime)
    publish.yaml       # config publisher: solo piattaforme abilitate per QUESTO podcast
    published.json     # stato pubblicazioni (committato dal workflow ad ogni run)
  <secondo-podcast>/   # in futuro: stessa struttura
.github/workflows/
  _pipeline.yml        # workflow riutilizzabile (workflow_call): restore→render→archive→publish→commit stato
  pensieriincodice.yml # caller: schedule + environment, invoca _pipeline
docs/PLAN.md           # questo file
```

### Multi-podcast e social parziali

- Ogni podcast ha la sua cartella `podcasts/<nome>/`.
- "Non tutti i podcast hanno tutti i social" è risolto dai flag `enabled` per
  piattaforma già presenti nel publisher: ogni podcast ha il suo `publish.yaml`.
- Aggiungere un podcast = una cartella + un caller di ~15 righe + un Environment.

### Secret per podcast → GitHub Environments

Ogni podcast ha canali/credenziali diverse. Un **Environment GitHub per podcast**
(es. `pensieriincodice`) con il suo set di secret. Il workflow riutilizzabile riceve
il nome dell'environment come input e ne eredita i secret. Restano cifrati anche su
repo pubblico.

Secret necessari per environment (podcast con YouTube + Telegram):

| Secret | Contenuto |
|---|---|
| `YOUTUBE_CLIENT_SECRETS` | contenuto JSON di `youtube_client_secrets.json` |
| `YOUTUBE_TOKEN` | contenuto JSON di `youtube_token.json` (contiene il refresh_token) |
| `TELEGRAM_API_ID` | api_id Telegram |
| `TELEGRAM_API_HASH` | api_hash Telegram |
| `TELEGRAM_SESSION_B64` | `base64` di `secrets/telegram.session` |

> La `telegram.session` dà accesso pieno all'account Telegram: è il secret più
> sensibile. Va **solo** in Environment secret, mai nel repo.

### Archivio video → GitHub Releases (una release per episodio)

- Tag per episodio, namespaced per podcast: `pensieriincodice-ep149`.
- Asset: uno `.zip` per episodio con la struttura `epNNN/sbN/...` (scaricabile a mano
  per la pubblicazione manuale; preserva i formati; consente il restore).
- Gli asset stanno **fuori** dal repo: la history git resta pulita, il repo snello.

### Persistenza su runner effimero

Il filesystem dei runner GitHub sparisce a fine job. La pipeline ricrea lo stato
del Mac così:

1. **restore**: scarica dalle Release recenti gli `.zip` in `./output`.
2. **generate**: il generator salta ciò che è già renderizzato (idempotente).
3. **archive**: ricarica/aggiorna gli `.zip` nelle Release.
4. **publish**: pubblica il prossimo soundbite non pubblicato.
5. **commit**: `published.json` viene committato di ritorno nel repo.

## Modalità pipeline (`mode`)

- `full` — restore → generate → archive → publish → commit
- `generate` — restore → generate → archive
- `publish` — restore → publish → commit

Migrazione fedele: `pensieriincodice.yml` esegue `generate` il martedì e `publish`
il giovedì (come i due workflow originali), ma su GitHub-hosted.

## Note e decisioni

- **Repo generator pinnato**: `valeriogalano/podcast-audiogram-generator` @ `main`
  (nome canonico da README/descrizione publisher). Nessun tag esiste sui tool →
  pin a `main`, comunque sovrascrivibile via input del workflow.
- **Font**: aggiunto blocco `fonts:` → DejaVu (preinstallato su `ubuntu-latest`),
  al posto del path macOS `/System/Library/Fonts/Helvetica.ttc`.
- **Path**: `output_dir`/`input_dir` resi relativi al workspace (erano su ProtonDrive).
- **Backlog**: con `restore` di default dell'ultima release + `episode: last`, il
  backlog coperto è quello dell'episodio più recente. Per drenare backlog su più
  episodi, aumentare `restore_releases` o allargare la selezione episodi.

## Fasi

### Fase 1 — Scaffolding (branch `feat/multi-podcast-github-hosted`) — IN CORSO
- [x] Salvare questo piano nel repo
- [x] `podcasts/pensieriincodice/{generate.yaml, publish.yaml}` migrati (font Linux, path relativi)
- [x] Spostare `published.json` in `podcasts/pensieriincodice/`
- [x] `_pipeline.yml` riutilizzabile
- [x] `pensieriincodice.yml` caller (generate mar / publish gio)
- [x] Rimuovere i vecchi `audiogram-generate.yml` / `audiogram-publish.yml`
- [x] Riscrivere il README (multi-podcast + come aggiungere un podcast)

### Fase 2 — Secret (richiede l'utente: le credenziali non sono accessibili all'agent)
- [ ] Creare l'Environment `pensieriincodice`
- [ ] Caricare i secret (lista sopra), inclusa la `telegram.session` in base64

### Fase 3 — Outward-facing (da confermare al momento)
- [ ] `gh repo rename podcast-audiogram-automation`
- [ ] Flip a **pubblico** (history verificata pulita: solo README + workflow)
- [ ] Primo run manuale (`workflow_dispatch`) per validare end-to-end

### Futuro
- [ ] Secondo podcast: cartella `podcasts/<nome>/` + caller + Environment
