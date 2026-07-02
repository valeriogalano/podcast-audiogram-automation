# podcast-audiogram-automation

Orchestratore **multi-podcast** per la generazione e pubblicazione automatica di
audiogram, su runner **GitHub-hosted** (nessun runner self-hosted).

Questo repo è il *control plane*: sa quali podcast esistono, quali feed, quali
social, quando girare e dove archiviare. Il lavoro pesante lo fanno due tool
generici e separati, richiamati dai workflow:

- [`podcast-audiogram-generator`](https://github.com/valeriogalano/podcast-audiogram-generator) — feed RSS → audiogram `.mp4`
- [`podcast-audiogram-publisher`](https://github.com/valeriogalano/podcast-audiogram-publisher) — `.mp4` → social

## Come funziona

Un workflow riutilizzabile (`.github/workflows/_pipeline.yml`) esegue, per un
singolo podcast, la pipeline su `ubuntu-latest`:

```
restore (Release) → generate → archive (Release) → publish → commit stato
```

- **restore** — scarica dalle GitHub Release recenti gli `.zip` in `./output`,
  così il runner effimero riparte dallo stato precedente e il generator è idempotente.
- **archive** — carica/aggiorna uno `.zip` per episodio come asset della Release
  `<podcast>-epNNN`. È l'**archivio scaricabile** anche per la pubblicazione manuale.
- **commit** — `published.json` (stato pubblicazioni) viene committato di ritorno.

Modalità (`mode`): `full` | `generate` | `publish`.

## Struttura

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

## Podcast configurati

| Podcast | Feed | Piattaforme | Schedule |
|---|---|---|---|
| Pensieri in codice | pensieriincodice.it | YouTube, Telegram | Mar 10:00 UTC (genera) · Gio 10:00 UTC (pubblica) |

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

## Aggiungere un podcast

1. Crea `podcasts/<nome>/` con `generate.yaml`, `publish.yaml`, `published.json` (`{}`).
2. Crea l'Environment GitHub `<nome>` e caricaci i secret delle sue piattaforme.
3. Duplica un caller (es. `pensieriincodice.yml`) come `<nome>.yml`, impostando
   `podcast: <nome>` e `environment: <nome>` e lo schedule desiderato.

Solo le piattaforme con `enabled: true` in `publish.yaml` vengono pubblicate: un
podcast può avere un sottoinsieme dei social.

## Esecuzione manuale

Dalla tab **Actions** → workflow del podcast → *Run workflow*, scegliendo `mode`.
