# pdpw-editorial — Design Spec

- Data: 2026-09-27
- Stato: DRAFT — in attesa di revisione
- Owner: pianic2
- Target: plugin Claude Code personale, repo separato `~/Documenti/GitHub/pdpw-editorial`

## 1. Obiettivo

Un profilo agente dedicato **esclusivamente** alla produzione di articoli per il portfolio
personale, che termina con una coppia di draft IT+EN in Wagtail tramite il PDPW MCP.

Successo significa:

- un comando (`/post <tema>`) porta un tema a `DRAFT_READY_FOR_HUMAN_PUBLICATION` senza interventi;
- ogni affermazione fattuale pubblicata è tracciabile a un Claim ID verificato e a ≥1 fonte nel ledger;
- la pipeline è riprendibile da qualunque fase fallita senza rifare le fasi valide;
- la scrittura su PDPW è idempotente e non può duplicare né sovrascrivere pagine;
- nessuna pubblicazione live automatica: la pubblicazione è sempre un'azione umana in Wagtail admin.

### Fuori scope (v1)

- aggiornamento di post esistenti (`update_page_draft`): la pipeline si ferma, non aggiorna;
- WordPress / Webflow;
- generazione immagini nativa (non disponibile in Claude Code): la featured image è opzionale;
- tipi di pagina diversi da `portfolio.BlogPostPage`;
- modifiche al backend PDPW (es. filtro `stable_id` su `list_pages`: eventuale task Jira PDPW separato).

## 2. Decisioni prese

| Decisione | Scelta |
|---|---|
| Posizione | Repo git separato, registrato come marketplace locale |
| Autonomia | End-to-end fino al draft PDPW; nessun gate umano intermedio |
| Knowledge base | File versionati nel plugin; Google Drive solo lettura opzionale |
| Architettura | 8 skill di fase + 1 agente orchestratore + contratto operativo |
| Stato terminale | `DRAFT_READY_FOR_HUMAN_PUBLICATION` |

## 3. Pipeline

```
BRIEF → RESEARCH/LEDGER → COVERAGE GAP → OUTLINE/DRAFT → FACT CHECK
      → EDITORIAL/SEO → EN LOCALIZATION → QA/PDPW DRAFT
```

| # | Skill | Fasi | Input | Output |
|---|---|---|---|---|
| 1 | `editorial-brief` | BRIEF | tema, `knowledge/*` | `01-brief.md` |
| 2 | `research-ledger` | RESEARCH + SOURCE LEDGER | 01 | `02-ledger.md`, `02-ledger.json` |
| 3 | `coverage-gap` | CONTENT GAP | 01, 02 | `03-gaps.md` |
| 4 | `outline-draft` | OUTLINE + DRAFT | 01, 02, 03 | `04a-outline.md`, `04b-draft.it.md` |
| 5 | `fact-check` | FACT CHECK | 02, 04b | `05-factcheck.md`, `05-factcheck.json` |
| 6 | `editorial-seo-review` | EDITORIAL REVIEW → SEO | 01, 02, 04b, 05 | `06-final.it.md`, `06-seo.it.json` |
| 7 | `localize-en` | LOCALIZATION | 01, 02, 03, 05, 06 | `07-final.en.md`, `07-seo.en.json` |
| 8 | `qa-publish-pdpw` | FINAL QA + PUBLISH | tutti | `08-qa.md`, `08-publish.json` |

Stato della pipeline: `pipeline.json`.

### 3.1 Skill — responsabilità

1. **editorial-brief** — applica i principi di `superpowers:brainstorming` in modalità autonoma
   (esplora e documenta alternative invece di porre domande). Produce tesi, audience, angolo,
   2–3 alternative scartate con motivazione, `article_type`, `stable_id`.
2. **research-ledger** — ricerca fonti (web search; Consensus se connesso; Drive in lettura se
   pertinente; evidenze di progetto: repo, commit, test, Jira, docs). Popola il ledger strutturato.
3. **coverage-gap** — revisione avversariale. Deve rispondere esplicitamente a:
   - What would invalidate the thesis?
   - What important counterargument is missing?
   - What would an expert criticize?
   - Which claim depends on outdated information?
   - What competing explanation exists?
   - What does the article imply without evidence?
   - What obvious reader question remains unanswered?

   Acumen è un enhancement, mai una dipendenza. Output minimo: ≥1 potenziale invalidatore della tesi.
   Ogni gap è marcato `ADDRESS | ACKNOWLEDGE | OUT_OF_SCOPE` con motivazione.
4. **outline-draft** — due checkpoint interni: `04a-outline.md` (struttura argomentativa, sezioni,
   claim previsti, gap indirizzati) poi `04b-draft.it.md` (prosa secondo `voice.md`). Ogni
   affermazione fattuale porta un marker `[C#]`.
5. **fact-check** — estrae i claim `[C#]`, li mappa alle fonti (N:N), assegna stato, classifica
   `critical`. Applica le regole di blocco (§5).
6. **editorial-seo-review** — PASS 1 editorial (accuratezza, argomentazione, leggibilità,
   terminologia, tono), poi PASS 2 SEO. Mai l'ordine inverso. SEO (Ahrefs se connesso, altrimenti
   SERP via web search) guida search intent, keyword, SERP gap, title, description, entità,
   internal linking — non riscrive l'argomentazione.
7. **localize-en** — genera un articolo EN **nuovo e semanticamente equivalente** da: significato
   IT finale + ledger + gap + audience EN + search intent EN. Non una traduzione frase per frase.
   Keyword EN ricercate indipendentemente dalle IT. Riusa gli stessi Claim ID.
8. **qa-publish-pdpw** — checklist QA, rimozione marker, costruzione payload, decisione di
   pubblicazione idempotente, una sola chiamata `create_localized_pair`, featured image opzionale.

## 4. Contratto dati

### 4.1 `stable_id`

- lowercase ASCII, kebab-case, regex `^[a-z0-9]+(-[a-z0-9]+)*$`, max 100 caratteri
  (compatibile con il vincolo PDPW `^[a-z0-9-]+$`);
- deterministico dal **soggetto** del brief, indipendente dal titolo editoriale finale;
- immutabile dopo la scrittura di `01-brief.md`;
- chiave logica tra filesystem, IT, EN, Wagtail, evidenze e revisioni future.

Esempio: tema "Come ho integrato OAuth 2.1 nel mio MCP server" → `oauth-2-1-mcp-server-integration`.

### 4.2 Header comune degli artefatti

Frontmatter YAML nei `.md`, campi top-level nei `.json`:

```yaml
pipeline_version: "1.0.0"
stable_id: oauth-2-1-mcp-server-integration
phase: 5
status: PASS | FAIL | BLOCKED
created_at: 2026-09-27T10:00:00Z
input_hash: sha256:<hex>   # sha256 degli artefatti di input dichiarati dalla fase (§3)
```

### 4.3 Article type e source policy

`article_type` è definito in `01-brief.md`:

| Tipo | Requisito minimo |
|---|---|
| `TECHNICAL` | ≥2 fonti primarie autorevoli (spec, docs ufficiali, codice sorgente) |
| `PROJECT_CASE_STUDY` | ≥1 fonte di evidenza di progetto + ≥1 artefatto implementativo; fonti esterne ufficiali dove applicabile |
| `RESEARCH` | ≥3 fonti ad alta autorevolezza; preferenza primarie / peer-reviewed |
| `OPINION` | evidenza richiesta per le premesse fattuali, non per giudizi personali dichiarati come tali |

Il mancato rispetto blocca a fine fase 2.

### 4.4 Ledger (`02-ledger.json`)

```yaml
sources:
  - source_id: S01                # ^S\d{2,}$
    url:
    title:
    publisher:
    author:
    published_at:
    accessed_at:
    source_type: primary | academic | authoritative-secondary | secondary | project-evidence
    authority: high | medium | low
    freshness: current | aging | stale
    notes:
    doi:                          # solo academic
    peer_reviewed:                # solo academic
```

`02-ledger.md` è la vista umana dello stesso contenuto. Il ledger è immutabile dopo la fase 2
(modificarlo invaliderebbe gli hash a valle). Per questo non contiene `claims_supported`: la
relazione N:N claim↔fonti vive solo in `05-factcheck.json`, e la vista source→claims è derivata
da lì (`pipeline.py status` la mostra).

### 4.5 Claim registry (`05-factcheck.json`)

```yaml
claims:
  - claim_id: C12                 # ^C\d+$
    text:
    sources: [S02, S05]           # N:N
    status: SUPPORTED | PARTIALLY_SUPPORTED | UNSUPPORTED | CONTRADICTED | STALE
    critical: true | false
    critical_reason:              # obbligatorio se critical
    action: KEEP | REVISE | REMOVE
    notes:
```

### 4.6 SEO (`06-seo.it.json`, `07-seo.en.json`)

```yaml
locale: it | en
title:            # ≤255
seo_title:        # ≤255
search_description:
slug:             # kebab-case, per-locale
excerpt:
primary_keyword:
secondary_keywords: []
search_intent:
entities: []
internal_links: []
evidence: []      # SERP / Ahrefs osservazioni, con fonte
```

### 4.7 `pipeline.json`

```yaml
pipeline_version:
stable_id:
created_at:
dry_run: false
phases:
  "1": {status: PASS, artifact_hashes: {...}, input_hash: ..., completed_at: ...}
  ...
terminal_state: null | DRAFT_READY_FOR_HUMAN_PUBLICATION | BLOCKED
blocker: null | {phase, code, detail}
```

### 4.8 `08-publish.json`

```yaml
decision: create | no-op | stop
content_hash: sha256:<hex>     # hash del payload IT+EN (senza marker)
payload: {type, stable_id, parent_stable_id: blog, it: {...}, en: {...}}
result: {it_page_id, en_page_id} | null
terminal_state:
```

## 5. Regole di fact checking

**Critical claim** = claim la cui falsità altererebbe materialmente:
la tesi dell'articolo; una raccomandazione tecnica; un'affermazione di sicurezza; una conclusione
quantitativa; l'attribuzione di un'azione o dichiarazione; informazioni di compatibilità/versione;
implicazioni legali o di sicurezza fisica.

| Stato | Critical | Non critical |
|---|---|---|
| `SUPPORTED` | KEEP | KEEP |
| `PARTIALLY_SUPPORTED` | REVISE obbligatorio (riformulare entro il supporto) | REVISE obbligatorio |
| `UNSUPPORTED` | **BLOCK** | REMOVE o riformulare come giudizio dichiarato |
| `CONTRADICTED` | **BLOCK** | REMOVE |
| `STALE` | **BLOCK** | REVISE con data/contesto o REMOVE |

Un BLOCK termina la pipeline con `terminal_state: BLOCKED` e `blocker` valorizzato.

## 6. Regole editoriali e SEO

- Editorial quality has precedence over keyword inclusion.
- Never alter a supported factual statement merely to improve SEO.
- In fase 6 il significato di un claim può cambiare solo applicando l'azione `REVISE`/`REMOVE`
  prescritta da `05-factcheck.json`; ogni altra modifica è solo di forma. Se la revisione
  editoriale richiede un claim nuovo o diverso, la fase 6 termina `FAIL` con codice
  `CLAIM_CHANGE_REQUIRED` e il resume riparte dalla fase 4 (nuovo 04b → nuovo fact-check).
- La fase 8 verifica che ogni `[C#]` in 06/07 esista in `05-factcheck.json` con azione ≠ `REMOVE`
  e che nessun claim `REMOVE` sia ancora presente.
- EN: nessun claim nuovo senza fonte nel ledger; ogni `[C#]` EN deve esistere in
  `05-factcheck.json` con azione `KEEP` o `REVISE` applicata.

## 7. Pubblicazione PDPW idempotente

Vincoli verificati sul server PDPW (2026-09-27):

- creazione solo via `create_localized_pair(type='portfolio.BlogPostPage', stable_id, parent_stable_id='blog', it, en)`, atomica, crea draft;
- nessun tool di pubblicazione live;
- `find_page` non cerca per `stable_id`; `list_pages` non filtra per `stable_id` (max 20 per pagina).

Lookup: paginazione `list_pages(type=['portfolio.BlogPostPage'], locale='it'|'en')`, confronto su `stable_id`.

| Stato remoto | Record locale `08-publish.json` | Decisione |
|---|---|---|
| nessuna pagina | — | `create` |
| coppia IT+EN trovata | stessi `page_id` e stesso `content_hash` | `no-op` |
| trovata | hash diverso, o nessun record locale | `STOP` (update workflow fuori scope v1) |
| più match / solo un locale / ambiguo | qualsiasi | `STOP` |

Rete di sicurezza server: il vincolo di unicità `(locale, stable_id)` rifiuta duplicati.
`--dry-run` calcola la decisione e il payload senza scrivere su PDPW.

## 8. Resumability

- Una fase è valida se `status: PASS` e il suo `input_hash` coincide con il ricalcolo corrente.
- `pipeline.py resume-point <stable_id>` restituisce la prima fase non valida; tutte le fasi a
  valle di una fase invalidata sono invalidate.
- Invalidazione esplicita: un `FAIL` con `rewind_to: <fase>` (es. `CLAIM_CHANGE_REQUIRED` →
  fase 4) invalida quella fase e le successive in `pipeline.json`, anche se gli hash coincidono.
  Il rewind conserva l'artefatto precedente come `<nome>.prev.md` come input della riscrittura.
- `/post resume <stable_id>` riparte da lì; `/post status <stable_id>` mostra la tabella fasi.
- `stable_id` e `01-brief.md` non vengono rigenerati su resume; cambiare tema = nuovo `stable_id`.

## 9. Profilo agente e allowlist

`agents/portfolio-editor.md`, campo `tools:` esplicito. Tutto ciò che non è elencato è escluso.

| Categoria | Consentiti |
|---|---|
| File | `Read`, `Write`, `Edit`, `Glob`, `Grep` — scrittura solo in `posts/<stable_id>/` (regola di contratto) |
| Shell | `Bash` — solo `uv run python plugins/pdpw-editorial/scripts/pipeline.py …` (regola di contratto) |
| Web | `WebSearch`, `WebFetch` |
| PDPW lettura | `list_pages`, `get_page`, `find_page`, `get_content_type_schema`, `list_images`, `get_image` |
| PDPW scrittura | `create_localized_pair`, `create_image` |
| Drive | `search_files`, `read_file_content` (sola lettura) |
| Skill | `Skill` (8 skill di fase + `superpowers:brainstorming`) |
| Opzionali | Consensus, Acumen, Ahrefs — slot documentato, abilitati quando connessi |

Esclusi esplicitamente: `update_page_draft`, `update_image`, `create_document`, `update_document`,
`Agent`, Atlassian, Calendar, Artifact, GitHub, scrittura Drive.

Connettori opzionali assenti: la skill usa il fallback e registra il gap in `evidence`/`notes`;
mai un blocco per assenza di un enhancement.

## 10. Condizioni di stop

La pipeline si ferma (`BLOCKED`) solo per:

- source policy non soddisfatta (fase 2);
- critical claim `UNSUPPORTED | CONTRADICTED | STALE` (fase 5);
- validazione schema fallita di un artefatto dopo un retry;
- decisione di pubblicazione `STOP` (fase 8);
- errore di validazione / autenticazione PDPW.

## 11. Struttura del repo

```
pdpw-editorial/
├── .claude-plugin/marketplace.json
├── pyproject.toml                       # uv; deps: jsonschema, pyyaml; dev: pytest, ruff
├── docs/specs/
├── plugins/pdpw-editorial/
│   ├── .claude-plugin/plugin.json
│   ├── CONTRACT.md                      # contratto operativo, fonte di verità per l'agente
│   ├── agents/portfolio-editor.md
│   ├── commands/post.md                 # /post <tema> | resume <id> | status <id> | --dry-run
│   ├── skills/<8 skill>/SKILL.md
│   ├── knowledge/{voice.md, editorial-policy.md, source-policy.md}
│   ├── schemas/{header,brief,ledger,factcheck,seo,publish,pipeline}.schema.json
│   └── scripts/pipeline.py              # validate | hash | status | resume-point | strip-markers | publish-decision | check-stable-id | check-source-policy
├── posts/<stable_id>/                   # versionato: archivio editoriale
└── tests/
```

## 12. Strategia di test

1. **Deterministici** (`uv run pytest -q`, `uv run ruff check .`) su `pipeline.py`:
   schema validation; hash e invalidazione a cascata; strip-markers; ogni ramo della tabella §7;
   regole `stable_id`; source policy per `article_type`; regole di blocco §5.
2. **Skill** (TDD di `superpowers:writing-skills`): per ogni skill uno scenario di pressione
   senza skill (RED) e con skill (GREEN). Minimi:
   - `fact-check` blocca un critical claim `CONTRADICTED`;
   - `editorial-seo-review` non altera un claim `SUPPORTED` per inserire una keyword;
   - `coverage-gap` produce ≥1 invalidatore della tesi;
   - `localize-en` non produce un calco frase per frase;
   - `editorial-brief` produce uno `stable_id` conforme e indipendente dal titolo;
   - `qa-publish-pdpw` sceglie `STOP` su pagina esistente senza record locale.
3. **End-to-end `--dry-run`** su tema reale (`PROJECT_CASE_STUDY`, OAuth 2.1 nel MCP server PDPW)
   fino a `08-publish.json`, senza scrittura su PDPW.
4. **Un run reale**: crea la coppia draft, verifica in Wagtail admin, rilancio → `no-op`.

Prerequisito per 3–4: ri-autenticazione del PDPW MCP (`/mcp`).

## 13. Rischi residui

- Le restrizioni su `Write` e `Bash` sono regole di contratto, non enforcement tecnico.
- Lookup `stable_id` via paginazione: O(n) chiamate; accettabile a volumi da portfolio.
- Qualità del fallback SEO senza Ahrefs: dati di volume assenti; dichiarato in `evidence`.
