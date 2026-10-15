<p align="center"><img src="assets/logo.png" alt="CapWizard" width="220"></p>

# CapWizard MCP

Il motore di [CapWizard](https://github.com/Cap-Wizard): da una cartella di video
genera un progetto CapCut già montato. **Il montaggio lo decide Claude**:
Whisper trascrive, poi l'AI sceglie ogni cosa e spiega perché.

Questo repository contiene il pacchetto Python `capwizard` (motore + CLI) e sarà la
base del **server MCP** che permette a Claude di creare e modificare progetti
CapCut direttamente. L'interfaccia grafica (app desktop) vive in un repository
separato e usa questo pacchetto come dipendenza.

> **Stato:** il server MCP non c'è ancora — per ora il motore si usa da CLI o
> come libreria (`capwizard.pipeline.run_pipeline`).

## Cosa fa

Generatore automatico di progetti CapCut per video social brevi.
Da una cartella di video genera un draft già pronto. **Il montaggio lo decide Claude**:
Whisper trascrive, poi l'AI sceglie ogni cosa e spiega perché.

- **guarda ogni clip** — dai fotogrammi, dall'audio e dalla luminosità misurata sul
  video capisce se è parlato rivolto al pubblico, b-roll o uno scarto; con un
  copione, a quale punto del copione corrisponde ogni ripresa («tapparelle su»,
  «luci accese»…) e in quale tratto l'azione accade davvero
- **regia** — prima di tagliare legge i testi: il copione, il parlato del girato e
  il video del template (i suoi sottotitoli, i testi a schermo, come usa i suoni).
  Da lì decide formato, tono, struttura e una consegna per ogni fase. Le
  indicazioni del copione sono **vincolanti**: quali riprese e in che ordine, la
  durata dei tagli («CUT da 0,5s» → ogni taglio dura 0,5s), la musica, i suoni, i
  testi a schermo. Ciò che il copione chiede e il materiale non ha (una ripresa
  mancante, l'audio del trend, un suono) finisce nell'elenco «DA FARE A MANO IN
  CAPCUT» in fondo al log
- **cosa tenere e cosa scartare** — dietro le quinte ("vai", "rifacciamo", "3 2 1"),
  frasi fuori tema, retake e ripetizioni (anche tra clip diverse), false partenze,
  filler, errori; legge insieme tutte le clip del progetto
- **b-roll** — se qualcuno parla, l'AI decide per ogni ripresa se va da sola a
  tutto schermo tra un blocco di parlato e l'altro (apertura, sequenze a tagli
  rapidi, stacchi) oppure sopra le frasi che illustra, su una traccia video muta;
  se non parla nessuno, costruisce un montaggio solo visivo (ordine, tratto e
  durata di ogni inquadratura). Parlato e b-roll possono stare nella stessa cartella
- **silenzi** — l'AI sceglie la soglia di pausa per il parlante e il ritmo; ogni
  silenzio tagliato viene registrato
- **testi** — titolo iniziale e testi di enfasi solo dove servono
- **suoni** — dai suoni del template, l'AI decide dove e se usarli (transizioni solo
  sui cambi di scena reali, enfasi solo sui momenti chiave) e se tenere la musica
- **kit del template** — i testi del template che non sono sottotitoli vengono letti a
  blocchi (titolo d'apertura, titoletto, elenco che entra voce per voce) insieme ai
  suoni che il template fa partire con ciascuno. A ogni video l'AI dice a cosa serve
  ogni blocco e lo riscrive, mantenendo font, colori, posizione e animazioni; colore
  ed effetti del template vengono estesi a tutto il video.
- **fine frase** (di default, `--no-sentence-transitions` per toglierlo) — sulla clip che
  chiude una frase l'inquadratura si stringe nell'ultimo secondo (100% → 115%), poi
  una transizione porta alla clip dopo con il woosh del template. La transizione è
  quella del template o, se non ne ha, quella che usi di più nei tuoi progetti CapCut
- **zoom** (opzionale, `--ai-zoom` o l'interruttore nell'app) — zoom in / zoom out
  leggeri (6–18%) sulle frasi chiave, scelti dall'AI; nel draft sono keyframe di
  scala come quelli fatti a mano in CapCut, e continuano attraverso i tagli dei silenzi
- **recensione finale** del montato con voto e suggerimento
- **sottotitoli identici al template** — ogni sottotitolo è una copia completa di
  uno vero del template (modello di testo, animazione, effetto, font, posizione),
  con il suo testo e i suoi tempi parola per parola; anche lunghezza delle frasi,
  maiuscole e punteggiatura seguono quelli del template
- stabilizzazione opzionale

**Senza Claude collegato il motore non genera** (nessun ripiego automatico).
Serve il CLI `claude` loggato (`claude` → `/login`) oppure `ANTHROPIC_API_KEY`.

Ogni scelta viene scritta nel log con il motivo e salvata in un **report Markdown**
(con la trascrizione annotata) in `~/Library/Application Support/CapWizard/reports/`.

## Setup

```bash
python3 -m venv .venv
.venv/bin/pip install -e .
```

Serve `ffmpeg`/`ffprobe` nel PATH (`brew install ffmpeg`).
Per la stabilizzazione di qualità superiore (opzionale): `brew install ffmpeg` con `vidstab`,
altrimenti il motore usa il filtro `deshake` integrato.

## CLI

```bash
# minimo
.venv/bin/capwizard /path/cartella --name "Reel 23"

# con template + ritmo serrato + durata target
.venv/bin/capwizard /path/cartella --name "Reel 23" \
  --template "COSA COMPRO CON 200K" --pacing fast --ai-target-duration 30

# modello Claude più accurato, copione di riferimento
.venv/bin/capwizard /path/cartella --name "Reel 23" \
  --template "COSA COMPRO CON 200K" --ai-model opus --script copione.docx
```

### Opzioni principali

- `--template, -t` — nome di un draft CapCut esistente. Effetti, overlay e stile sottotitoli sono ereditati; i suoni del template li gestisce l'AI.
- `--ai-model` — `sonnet` (default) / `opus` / `haiku`: il modello Claude che monta.
- `--ai-target-duration` — durata a cui l'AI deve puntare (default: la decide l'AI).
- `--pacing` — `normal` / `fast` (default) / `aggressive`: ritmo desiderato, passato all'AI.
- `--script` — copione (txt/md/docx): l'AI ne ricava la regia (struttura, indicazioni
  tra parentesi) e lo usa per capire cosa è fuori copione. Le durate scritte nel
  copione vengono ignorate: la durata la decidi solo tu con `--ai-target-duration`.
- `--no-fillers` / `--aggressive-fillers` — cosa chiedere all'AI sugli intercalari.
- `--no-ai-sounds` — lascia l'audio del template com'era, senza decisioni dell'AI.
- `--no-emphasis` / `--emphasis-count` — testi di enfasi (default max 4, li sceglie l'AI).
- `--intro-title` — titolo iniziale scelto da te (altrimenti lo sceglie l'AI).
- `--no-ai-review` — salta la recensione finale.
- `--stabilize` — stabilizza i video con vidstab/deshake (risultati in cache).
- `--manual-subtitles` — non scrivere i sottotitoli come quelli del template: la
  dimensione dei blocchi la danno `--subtitle-max-words` / `--subtitle-max-duration`
  (lo stile resta quello del template).
- `--model` — Whisper: `tiny|base|small|medium|large-v3` (default `large-v3`).
- `--language` — codice lingua (default `it`).

Tutto il resto del template (overlay video, filtri, regolazioni colore, text overlay non-sottotitoli) viene preservato; ciò che cade oltre la nuova durata del video viene troncato/scartato.

## Architettura

```
src/capwizard/
├── cli.py          # entry point CLI (click)
├── pipeline.py     # orchestratore: AI check → trascrizione → AI guarda le clip → regia → montaggio / b-roll → sottotitoli → testi/suoni → draft → recensione
├── ai.py           # Claude: look_at_clips / plan_brief / plan_edit / plan_covers / plan_montage / plan_texts / plan_sounds / review (CLI o SDK, nessun fallback)
├── vision.py       # strisce di fotogrammi per far vedere le clip all'AI
├── edit.py         # trascrizione globale multi-clip, applica le decisioni AI, taglia i silenzi, log
├── report.py       # log live + report Markdown per progetto
├── registry.py     # registro dei progetti generati (app-data)
├── probe.py        # wrapper ffprobe
├── transcribe.py   # faster-whisper, timestamp per parola
├── cuts.py         # preset di ritmo (pad ai bordi delle frasi)
├── subtitles.py    # blocchi di sottotitoli: a mano o a frasi come il template
├── emphasis.py     # testi di enfasi
├── cleanup.py      # filler, retake, false partenze
├── stabilize.py    # ffmpeg vidstab + deshake di riserva, con cache
├── draft.py        # scrittura draft CapCut (da zero)
├── template.py     # scrittura draft CapCut da template: clone dei sottotitoli, lettura del template per la regia, piano suoni AI
└── models.py       # dataclass
```

## Note di fragilità

Il formato `.draft` di CapCut non è ufficiale. Il template-mode è meno fragile
del from-scratch: clona lo schema da un draft esistente prodotto dalla tua
versione di CapCut. Se aggiorni CapCut e qualcosa rompe, di solito basta
ricreare il draft template nella nuova versione di CapCut e rilanciare.
