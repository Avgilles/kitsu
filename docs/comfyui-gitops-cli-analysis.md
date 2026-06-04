# `comfyui-cli` — Analyse & specs d'un CLI GitOps pour ComfyUI

> Document d'analyse et de conception. Objectif : capturer le « knowledge » du
> projet [`n8n-cli`](https://github.com/edenreich/n8n-cli) et le transposer en
> un outil équivalent pour **ComfyUI**, en **Python**.
>
> **Conclusion clé (à lire avant tout) :** il ne faut **pas** copier n8n-cli
> à l'identique. En ComfyUI, le fichier JSON du workflow est la partie « pas
> chère » ; la vraie douleur est la **dérive de dépendances** (nœuds custom +
> modèles référencés par *nom* et non par *hash*). Le créneau réellement libre
> est un CLI **déclaratif de réconciliation de l'état désiré** (workflows +
> dépendances) avec **détection de drift**, façon Argo CD / Flux — pas un
> simple `sync` de JSON.

---

## Sommaire

1. [Comment fonctionne `n8n-cli`](#1-comment-fonctionne-n8n-cli)
2. [Analyse de `comfy-pilot` (ce qui est réutilisable)](#2-analyse-de-comfy-pilot)
3. [La différence de domaine n8n ↔ ComfyUI](#3-la-différence-de-domaine-n8n--comfyui)
4. [État de l'art : outils existants pour ComfyUI](#4-état-de-lart--outils-existants-pour-comfyui)
5. [Le créneau libre (gap analysis)](#5-le-créneau-libre-gap-analysis)
6. [Design proposé pour `comfyui-cli` (Python)](#6-design-proposé-pour-comfyui-cli-python)
7. [Choix techniques & dépendances](#7-choix-techniques--dépendances)
8. [Roadmap & risques](#8-roadmap--risques)
9. [Sources](#9-sources)

---

## 1. Comment fonctionne `n8n-cli`

CLI **100 % Go** (framework **Cobra**, build via **Taskfile**). Philosophie :
**GitOps** — *« piloter n8n depuis des fichiers versionnés dans Git »*. Les
workflows ne sont plus édités à la main dans l'UI mais stockés comme fichiers
(JSON/YAML) dans un repo, puis **synchronisés** vers l'instance n8n.

### Architecture (couches séparées)

```
main.go            → point d'entrée, lance le rootCmd
cmd/               → définition Cobra (workflows.go = commande parente + sous-commandes)
config/            → lecture des variables d'env / .env (N8N_API_KEY, N8N_INSTANCE_URL)
n8n/               → cœur métier : client HTTP REST + logique de sync
logger/            → sortie/logs
openapi.yml        → spec de l'API n8n
install.sh         → install auto (détecte OS/arch)
examples/          → workflows d'exemple + intégration GitHub Actions
```

Point de design central : **`cmd/` ne contient que du « glue » Cobra**
(parsing de flags + affichage) ; toute la logique réelle vit dans le package
`n8n/`. C'est ça qui rend le code testable et propre.

### Commandes

| Commande | Rôle |
|----------|------|
| `workflows list` | Liste les workflows distants (sortie `table`/`json`/`yaml`, limite ≤ 250) |
| `workflows refresh` | **Pull** : récupère l'état distant → écrit/maj les fichiers locaux (`--dry-run`) |
| `workflows sync` | **Push** : applique les fichiers locaux → instance (`--prune`, `--dry-run`) |
| `workflows activate` / `deactivate` | Active/désactive un workflow par ID |
| `version` | Version du CLI |

### Le mécanisme central (« secret sauce »)

`sync` fait un **diff déclaratif** entre l'état local et l'état distant, basé
sur l'**ID du workflow** :

- workflow avec **ID existant** → `UPDATE` (PUT)
- workflow **sans ID / absent du distant** → `CREATE` (POST), l'ID renvoyé est
  ré-injecté dans le fichier local
- distant présent mais **absent en local + `--prune`** → `DELETE`

C'est exactement le modèle **`kubectl apply` / Terraform** : *l'état désiré =
les fichiers Git*, le CLI réconcilie. `refresh` est l'opération inverse
(importer l'existant dans Git pour démarrer).

> ⚠️ Ce modèle **repose sur deux prérequis n8n** qui n'existent PAS en
> ComfyUI : (a) une **API CRUD de workflows stockés** avec des **IDs stables**,
> et (b) un workflow JSON **auto-suffisant**. Voir §3.

---

## 2. Analyse de `comfy-pilot`

[`AdamPerlinski/comfy-pilot`](https://github.com/AdamPerlinski/comfy-pilot) —
**Python (88 %) + JS (12 %)**, **licence GPL-3.0**.

**Ce n'est PAS un CLI ni du GitOps.** C'est une **extension/custom node** *dans*
l'UI ComfyUI : un **copilote IA conversationnel** qui génère des workflows à
partir de langage naturel (9 backends : Ollama, Claude Code, Gemini, OpenAI…),
exposé via 4 endpoints maison (`/comfy-pilot/chat`, `/agents`, `/system`,
`/models`). Le JSON généré est inséré dans le canvas, l'utilisateur lance la
queue **à la main**.

### Ce qui est réutilisable pour nous (inspiration *structure de code*, pas paradigme)

| Brique | Fichiers | Intérêt pour `comfyui-cli` |
|--------|----------|----------------------------|
| **Base de connaissance** | `knowledge/*.md` + `knowledge/*.py` (`core_nodes.md`, `custom_nodes.md`, `models.md`, `workflow_patterns.md`, `workflow_tuning.md`, `manager.py`) | Format double markdown + module Python. Brique idéale pour **valider** qu'un workflow a ses nœuds/modèles + estimer le VRAM. |
| **Manipulation de workflow** | `workflow/templates.py`, `workflow/manipulator.py` | Construire/modifier le JSON par programme. |
| **Pattern providers** | `agents/base.py` (classe abstraite `AgentBackend`) + `agents/registry.py` (auto-enregistrement) | Transposable à un système de « providers » (instances ComfyUI, backends de stockage de modèles…). |

> ⚠️ **Contrainte de licence** : `comfy-pilot` est **GPL-3.0**. Réutiliser son
> code obligerait notre CLI à être GPL-3.0 aussi. À trancher tôt (cf. §7).

---

## 3. La différence de domaine n8n ↔ ComfyUI

C'est le point critique. Les deux manipulent des « workflows » (graphes de
nœuds JSON), mais leurs API ont des **finalités opposées**.

| | **n8n** | **ComfyUI** |
|---|---|---|
| Nature du workflow | Automatisation persistante (activée, déclenchée par webhook/cron) | Pipeline de **génération d'image** exécuté à la demande |
| API « workflows » | **CRUD REST** sur des workflows stockés en DB, avec **IDs stables**, versioning, activation | **Aucune.** Pas d'entité « workflow » serveur, pas d'ID, pas d'activation |
| Stockage serveur | n8n est la source de vérité (DB) | Fichiers sur disque (`user/default/workflows/*.json`), édités côté navigateur |
| « Activer » un workflow | Concept central | N'existe pas |
| Workflow auto-suffisant ? | Oui (JSON + stubs de credentials) | **Non** : le JSON ne contient ni le code des custom nodes, ni les modèles, ni les chemins locaux |

### 3.1 Deux formats JSON, un seul exécutable

C'est le piège n°1 pour qui conçoit un outil de sync.

- **Format UI / éditeur** (« workflow ») — sérialisation LiteGraph du canvas
  complet : `nodes[]`, `links[]`, `groups[]`, `pos`, `size`, `widgets_values`
  (tableau positionnel)… C'est ce qui est **sauvegardé sur disque** et rechargé
  par l'éditeur. **Non exécutable** via l'API. Spec :
  <https://docs.comfy.org/specs/workflow_json>
- **Format API / prompt** (« Save (API Format) », mode dev) — objet **indexé
  par node id** : `{ "<id>": { "class_type": ..., "inputs": {...}, "_meta": {...} } }`,
  les connexions étant des références `["<sourceId>", <slot>]`. **C'est le seul
  format accepté par `POST /prompt`.**

**Conversion :**
- UI → API se fait **uniquement côté client** (fonction `graphToPrompt` dans le
  frontend TS, qui s'appuie sur l'ordre des inputs fourni par `/object_info`).
  **Aucun endpoint serveur** ne le fait — feature request ouverte/non
  implémentée : <https://github.com/comfyanonymous/ComfyUI/issues/1112>
- API → UI **n'est pas reconstructible** sans perte (le format API a jeté toute
  la donnée de layout). Workaround historique : ComfyUI embarque le workflow UI
  dans les métadonnées PNG des images générées.

> **Implication majeure de design.** Si on versionne du **format API**, on peut
> exécuter mais on perd la vue éditeur (et il n'y a pas de retour officiel vers
> l'UI). Si on versionne du **format UI**, c'est fidèle à ce que voit l'humain
> mais **non exécutable** sans réimplémenter `graphToPrompt` + `/object_info`.
> Pour faire les deux, **on doit posséder notre propre convertisseur UI→API**.

### 3.2 L'API ComfyUI pertinente

Endpoints confirmés dans `server.py` / `app/user_manager.py` :

| Endpoint | Rôle |
|----------|------|
| `POST /prompt` | Met un graphe **format API** en file → `{prompt_id, number, node_errors}` |
| `GET /history/{prompt_id}` | Résultat d'un run : outputs `{images:[{filename, subfolder, type}]}` |
| `GET /queue` / `POST /queue` | État de file ; `{"clear":true}` ou `{"delete":[...]}` |
| `POST /interrupt`, `POST /free` | Annuler le job courant / décharger les modèles |
| `GET /view?filename=&subfolder=&type=output` | Télécharge un fichier de sortie |
| `GET /object_info` | **Schéma de tous les nœuds** (inputs requis/optionnels + ordre, types, défauts) — base de la validation/conversion |
| `POST /upload/image`, `/upload/mask` | Pousser des assets d'entrée |
| `GET /ws` (WebSocket) | Progression/exécution temps réel (fin = message `executing` avec `node:null`) |
| **`/userdata/*`** | **Store de fichiers générique** (GET liste, GET/POST/DELETE/move). C'est ce que le frontend utilise pour sauver/charger les workflows **UI**. |

> Le `/userdata` est le plus proche analogue d'une « API de workflows », mais
> c'est un **store de fichiers indexé par chemin** : pas d'ID stable, pas de
> versioning, pas de validation que le blob est un workflow. Le contenu y est en
> **format UI** (non exécutable tel quel).

---

## 4. État de l'art : outils existants pour ComfyUI

| Outil | URL | Langage | Catégorie | Vrai GitOps de workflows ? | Maturité |
|-------|-----|---------|-----------|----------------------------|----------|
| **comfy-cli** (officiel) | github.com/Comfy-Org/comfy-cli | Python | Installeur/manager (+`comfy run`, `comfy generate`) | **Non** (lockfile `comfy-lock.yaml` en beta, mais impératif, pas de réconciliation) | ~835★, très actif, GPL-3.0 |
| ComfyUI-Deploy / ComfyDeploy | github.com/BennyKok/comfyui-deploy | TS/Py | Hébergement/API « Vercel for ComfyUI » | Non (versioning interne à la plateforme) | ~1.5k★, WIP, AGPL-3.0 |
| ViewComfy | viewcomfy.com | (SaaS) | Upload `workflow_api.json` → API | Non | Commercial |
| RunPod / Modal / Replicate / Fal | runpod-workers/worker-comfyui | Py | Backend d'exécution serverless | Non (le JSON est un payload) | Mature |
| comfy-pack (BentoML) | github.com/bentoml/comfy-pack | Py | Artefact reproductible `.cpack.zip` + API | Non (zip opaque, impératif) | ~217★, Apache-2.0 |
| **ComfyGit** | github.com/comfygit-ai/comfygit | Py | Versioning d'**environnements** | **Le plus proche** — git comme source de vérité, mais pour l'*env* (pas les workflows), sans boucle de réconciliation | ~19★, v0.5, très tôt |
| ComfyUI-Manager `cm-cli` snapshots | github.com/Comfy-Org/ComfyUI-Manager | Py | Save/restore de l'env (nœuds) | Non, mais **primitive « apply » la plus proche** (env only) | Mature, officiel |
| Workspace Manager (11cafe) | github.com/11cafe/comfyui-workspace-manager | TS | Historique de versions interne | Non (pas git) — **déprécié** | ~1.4k★, mort |
| ComfyRun / ComfyWorkflows | github.com/thecooltechguy/ComfyUI-ComfyRun | Py/JS | Partage/run one-shot | Non (explicitement « pas du GitOps ») | ~82★ |

### Bibliothèques/SDK API (fondations possibles, en Python)

| Lib | URL | Langage | Note |
|-----|-----|---------|------|
| **sugarkwork/Comfyui_api_client** | github.com/sugarkwork/Comfyui_api_client | Python (sync+async) | Le plus généraliste : prompt, history, WS, upload, **conversion auto de format**, lookup de nœuds. MIT. |
| **deimos-deimos/comfy_api_simplified** | github.com/deimos-deimos/comfy_api_simplified | Python | Éditer un workflow **format API**, set de params, queue, récup images. ~112★. |
| comfyorg/comfyscript | github.com/comfyorg/comfyscript | Python | DSL haut niveau (Python ↔ nœuds), plus lourd qu'un client. |
| (réf.) `websockets_api_example.py` | dépôt officiel ComfyUI | Python | LA référence du flux complet POST /prompt + WS. |

> **Pour un CLI Python**, base recommandée : `sugarkwork/Comfyui_api_client`
> (MIT, sync+async, conversion de format incluse) — ou wrapper directement
> l'exemple officiel. (NB : aucun client **Go** mature n'existe, ce qui
> renforce le choix Python.)

---

## 5. Le créneau libre (gap analysis)

**Personne ne fait de vrai GitOps déclaratif de workflows ComfyUI.** L'écosystème
se répartit en trois catégories, dont **aucune** n'est ce que vise n8n-cli :

1. **Versioning manuel du JSON** (`git commit` des fichiers, pas d'apply auto).
2. **Reproductibilité d'environnement** (versionne l'*env* : nœuds, modèles,
   deps — pas la logique du workflow). → ComfyGit, cm-cli snapshots, comfy-pack.
3. **Hébergement / endpoint API** (emballe un `workflow_api.json` en API ;
   versioning interne à la plateforme).

**Verdict sur les 4 pistes envisagées :**

- **(a) Sync/versioning des fichiers de workflow** — *la plus faible.* Le JSON
  est déjà du texte versionnable ; copier le modèle « sync JSON → instance » de
  n8n-cli échoue car **la valeur de ComfyUI vit en-dehors du JSON**.
- **(b) Exécution reproductible + download des sorties** — *banalisée.* Déjà
  couverte par `comfy run` et une douzaine de wrappers.
- **(c) Réconciliation de dépendances** (garantir nœuds custom + modèles
  présents) — *la vraie douleur*, répétée partout (« pas d'équivalent Docker /
  lockfile en ComfyUI »), **partiellement** occupée par `comfy-lock.yaml`,
  comfy-pack, snapshots.
- **(d) Autre** → la synthèse ci-dessous.

> **Le créneau réellement libre = (c)+(a) fait de façon déclarative et
> continue, comme Argo CD / Flux pour Kubernetes.** Aucun outil existant ne le
> fournit.

### Le produit défendable

Un CLI qui traite un **repo Git comme la source de vérité unique de l'état
désiré ComfyUI** — workflow(s) + un **lockfile** épinglant les **SHA des
commits des custom nodes** + **hashes/sources des modèles** + deps Python — puis
fait de la **réconciliation idempotente** contre une instance (locale ou
distante) :

1. **Déclaratif + réconciliant**, pas impératif (vs `comfy-cli`/`comfy-pack` qui
   imposent des étapes pack/restore manuelles).
2. **Git-natif**, pas un zip opaque (vs `.cpack.zip` non diffable en PR).
3. **Neutre vis-à-vis du provider / local-first** (vs ViewComfy/Replicate, lock-in
   cloud ; vs comfy-deploy, orienté Modal).
4. **Détection de drift en commande de premier rang** (`status`/`diff`) — révèle
   le **mismatch silencieux de version de modèle** (même nom de fichier, poids
   différents) qui corrompt les sorties **sans erreur**. C'est la feature la plus
   précieuse et la moins servie.

### Risques à valider AVANT de coder (cf. §8)

- **Comfy-Org peut absorber ce créneau** : `comfy-lock.yaml` existe déjà en
  beta ; ajouter une boucle de réconciliation est « à une feature près ».
- **La distribution des modèles est la partie dure** : épingler un hash est
  facile ; *récupérer* de façon fiable des poids multi-Go gated (auth HF,
  Civitai, buckets privés) à travers un parc, c'est là qu'est le vrai travail.

---

## 6. Design proposé pour `comfyui-cli` (Python)

### 6.1 Positionnement

> **`comfyui-cli` = un outil GitOps déclaratif de l'état désiré ComfyUI.**
> Cœur de valeur : **réconciliation de dépendances + détection de drift**.
> L'exécution de workflows est une **commande de confort**, pas le cœur.

On garde l'**architecture** et la **philosophie déclarative** de n8n-cli
(pull/push, `--dry-run`, `--prune`), mais le cœur métier change : on ne
réconcilie pas un CRUD de workflows, on réconcilie un **état** (workflows +
nœuds + modèles).

### 6.2 Le fichier d'état désiré : `comfyui.lock.yaml`

Source de vérité versionnée dans Git, à côté des workflows :

```yaml
# comfyui.lock.yaml — état désiré, commité dans Git
version: 1
instance: ${COMFYUI_INSTANCE_URL}      # résolu depuis l'env

workflows:
  - path: workflows/portrait_sdxl.json # format UI (éditable) OU api/
    format: ui                          # ui | api
    # le CLI dérive l'empreinte exécutable (graphToPrompt) à la volée

custom_nodes:
  - name: ComfyUI-Impact-Pack
    repo: https://github.com/ltdrdata/ComfyUI-Impact-Pack
    commit: 9f2c1ab                     # SHA épinglé → reproductible

models:
  - name: sd_xl_base_1.0.safetensors
    type: checkpoints
    source: huggingface:stabilityai/...
    sha256: 31e35c80fc...               # ← clé anti-drift silencieux
```

### 6.3 Architecture des modules (Python)

```
comfyui_cli/
  __main__.py            # entrée, construit le groupe Click/Typer
  cli/                   # "glue" CLI uniquement (parsing + affichage) — cf. n8n-cli cmd/
    workflow.py          # run / list / validate / convert
    state.py             # apply / diff / status / pull
    node.py              # list / install / sync
    model.py             # list / pull / verify
    queue.py             # status / cancel / clear
    output.py            # download / watch
  core/                  # cœur métier (TOUTE la logique, testable)
    client.py            # client HTTP+WS ComfyUI (sur base sugarkwork ou maison)
    reconciler.py        # diff état désiré (lock) ↔ état réel (instance) + convergence
    drift.py             # détection de drift (hash modèles, SHA nœuds)
    convert.py           # convertisseur UI → API (port de graphToPrompt + /object_info)
    lockfile.py          # parsing/écriture de comfyui.lock.yaml
  providers/             # pattern base+registry inspiré de comfy-pilot agents/
    models/              # huggingface / civitai / url / s3 ...
  knowledge/             # (optionnel) md+py inspiré de comfy-pilot pour la validation
  config.py              # COMFYUI_INSTANCE_URL, COMFYUI_API_KEY (.env)
  logger.py
tests/
install.sh
pyproject.toml
```

Règle d'or (héritée de n8n-cli) : **`cli/` ne contient que du parsing/affichage ;
tout le métier est dans `core/`.**

### 6.4 Commandes (mapping depuis n8n-cli)

| n8n-cli | `comfyui-cli` équivalent | Notes |
|---------|--------------------------|-------|
| `workflows sync` | `apply` | **Réconcilie** le lockfile → instance : installe nœuds (SHA), fetch modèles (hash), valide les workflows. `--dry-run`, `--prune`. |
| `workflows refresh` | `pull` | Importe l'état réel (workflows via `/userdata`, nœuds/modèles installés) → lockfile + fichiers. |
| `workflows list` | `workflow list` / `node list` / `model list` | via `/userdata`, `/object_info`. |
| `activate`/`deactivate` | `queue cancel` / `queue clear` | la file remplace l'« activation ». |
| — (nouveau) | **`status` / `diff`** | **Détection de drift** : désiré ↔ réel. *La commande phare.* |
| — (nouveau) | `workflow run <file>` | `POST /prompt` (+ conversion UI→API) + suivi WS + download. *Confort.* |
| — (nouveau) | `workflow validate <file>` | Vérifie nœuds requis vs `/object_info` et modèles requis vs disque. |
| — (nouveau) | `workflow convert <ui.json> -o api.json` | Expose notre convertisseur. |

### 6.5 Flux type (GitOps)

```bash
# 1. Importer un état existant dans Git
comfyui pull --instance http://gpu-box:8188 -o comfyui.lock.yaml
git add comfyui.lock.yaml workflows/ && git commit -m "Snapshot initial"

# 2. Vérifier la dérive (CI ou local)
comfyui status            # → "model sd_xl_base_1.0: hash mismatch (drift!)"

# 3. Réconcilier une autre machine vers l'état désiré
comfyui apply --dry-run   # prévisualise
comfyui apply             # installe nœuds@SHA, fetch modèles@hash, valide

# 4. Exécuter (confort)
comfyui workflow run workflows/portrait_sdxl.json --output ./out
```

---

## 7. Choix techniques & dépendances

| Sujet | Décision proposée | Justification |
|-------|-------------------|---------------|
| Langage | **Python 3.10+** | Choix utilisateur ; écosystème ComfyUI 100 % Python ; aucun client Go mature. |
| Framework CLI | **Typer** (ou Click) | Équivalent Pythonic de Cobra ; sous-commandes, help, complétion. |
| Client API | Base **`sugarkwork/Comfyui_api_client`** (MIT) ou client maison léger | Conversion de format incluse ; MIT compatible. |
| Packaging | `pyproject.toml` + `pipx`/`uv`, `install.sh` auto | Calque la distribution de comfy-cli. |
| Config | `.env` + variables d'env (`COMFYUI_INSTANCE_URL`, `COMFYUI_API_KEY`) | Calque n8n-cli ; **ne jamais committer le `.env`**. |
| Licence | **MIT/Apache-2.0** recommandé | ⚠️ **Ne pas copier le code GPL de `comfy-pilot`** (le `knowledge/` est ré-implémentable). `comfy-cli` est GPL-3.0 → s'en inspirer sans copier. |
| Convertisseur UI→API | À **porter** depuis `graphToPrompt` (frontend TS) | Pas d'endpoint serveur ; nécessaire si on versionne du format UI. |

---

## 8. Roadmap & risques

### MVP (le strict nécessaire pour valider la valeur)
1. `config` + `core/client.py` (HTTP `/prompt`, `/history`, `/view`, `/object_info`, WS).
2. `workflow run` + `workflow validate` (confort + preuve que le client marche).
3. `core/convert.py` (UI→API) — débloque tout le reste.

### V1 (le cœur de valeur)
4. `core/lockfile.py` + `pull` (snapshot de l'état réel).
5. `core/drift.py` + **`status`/`diff`** (la feature phare).
6. `apply` + `providers/models/*` (réconciliation : nœuds@SHA, modèles@hash).

### Risques
- **Absorption par Comfy-Org** : `comfy-lock.yaml` (beta) est l'endroit
  naturel pour ajouter la réconciliation → notre différenciation peut s'éroder.
  *Mitigation : viser la boucle de réconciliation + drift, là où ils sont
  absents aujourd'hui ; rester compatible/importable depuis `comfy-lock.yaml`.*
- **Distribution des modèles** : fetch fiable de poids multi-Go gated (HF auth,
  Civitai, buckets) = le vrai travail d'ingénierie. *Mitigation : pattern
  `providers/` extensible, commencer par HF + URL directe.*
- **Convertisseur UI→API** : doit suivre l'évolution du frontend ComfyUI.
  *Mitigation : sinon, n'accepter que le format API en entrée pour le MVP.*
- **GPU en CI** : la validation par exécution réelle exige un runner GPU
  self-hosted ; `validate` (statique, sans run) contourne ce besoin pour la CI.

---

## 9. Sources

**n8n-cli & comfy-pilot**
- <https://github.com/edenreich/n8n-cli>
- <https://github.com/AdamPerlinski/comfy-pilot>

**CLI & outils ComfyUI**
- <https://github.com/Comfy-Org/comfy-cli> (+ `comfy-lock.yaml` beta)
- <https://github.com/BennyKok/comfyui-deploy> · <https://github.com/comfy-deploy/comfydeploy>
- <https://github.com/bentoml/comfy-pack>
- <https://github.com/comfygit-ai/comfygit>
- <https://github.com/Comfy-Org/ComfyUI-Manager> (`cm-cli`, snapshots)
- <https://github.com/11cafe/comfyui-workspace-manager> (déprécié)
- <https://github.com/thecooltechguy/ComfyUI-ComfyRun>
- <https://github.com/SaladTechnologies/comfyui-api>

**Clients / SDK API**
- <https://github.com/sugarkwork/Comfyui_api_client>
- <https://github.com/deimos-deimos/comfy_api_simplified>
- <https://github.com/comfyorg/comfyscript>
- <https://github.com/comfyanonymous/ComfyUI/blob/master/script_examples/websockets_api_example.py>

**Formats & API serveur ComfyUI**
- <https://docs.comfy.org/specs/workflow_json> (format UI)
- <https://docs.comfy.org/development/comfyui-server/comms_routes> (routes serveur)
- <https://github.com/comfyanonymous/ComfyUI/blob/master/script_examples/basic_api_example.py> (format API)
- <https://github.com/comfyanonymous/ComfyUI/issues/1112> (pas de convertisseur serveur UI→API)
- `comfyanonymous/ComfyUI/server.py`, `app/user_manager.py` ; `Comfy-Org/ComfyUI_frontend/src/utils/executionUtil.ts` (`graphToPrompt`), `src/scripts/api.ts` (`/userdata`)

**Reproductibilité / drift (pain points)**
- <https://www.numonic.ai/blog/sharing-comfyui-workflows-team> (« pas d'équivalent Docker/lockfile »)
- <https://eastondev.com/blog/en/posts/ai/20260602-comfyui-workflow-reuse-guide/>
- <https://www.magnopus.com/blog/sharing-models-and-custom-nodes-in-comfyui>
</content>
</invoke>
