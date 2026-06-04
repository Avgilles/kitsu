# Dépendances Python ComfyUI, ComfyGit & l'orchestrateur

> Document complémentaire à [`comfyui-gitops-cli-analysis.md`](./comfyui-gitops-cli-analysis.md).
>
> **Sujet :** pourquoi la gestion des dépendances Python est *le* point de
> douleur de ComfyUI, comment **ComfyGit** s'y attaque, et **où cette couche
> s'insère dans l'orchestrateur GitOps** qu'on conçoit.
>
> **Thèse centrale :** en ComfyUI, *« faire tourner un workflow » = JSON +
> environnement Python reproductible (nœuds @commit + paquets pip résolus +
> build torch/CUDA + modèles @hash)*. Le JSON est la partie facile. **La couche
> d'install/dépendances EST le cœur de l'orchestrateur**, pas un détail
> d'implémentation. ComfyGit résout ~80 % de cette couche ; il lui manque
> précisément ce que l'orchestrateur doit apporter (réconciliation continue +
> détection de drift + lockfile bit-à-bit).

---

## Sommaire

1. [Pourquoi les dépendances Python cassent ComfyUI](#1-pourquoi-les-dépendances-python-cassent-comfyui)
2. [Comment ComfyGit s'y attaque](#2-comment-comfygit-sy-attaque)
3. [Le paysage des solutions (comparatif)](#3-le-paysage-des-solutions-comparatif)
4. [Ce que ComfyGit ne fait PAS (les trous)](#4-ce-que-comfygit-ne-fait-pas-les-trous)
5. [Lien avec l'orchestrateur : où ComfyGit s'insère](#5-lien-avec-lorchestrateur--où-comfygit-sinsère)
6. [Trois scénarios d'intégration](#6-trois-scénarios-dintégration)
7. [Recommandation](#7-recommandation)
8. [Sources](#8-sources)

---

## 1. Pourquoi les dépendances Python cassent ComfyUI

### 1.1 Un seul venv partagé + un `requirements.txt` par nœud = « dependency hell »

ComfyUI fait tourner **le cœur ET tous les custom nodes dans un seul
environnement Python**. Chaque custom node embarque son propre
`requirements.txt`, et installer un nœud **mute l'environnement global** pour
tous les autres. Les mainteneurs eux-mêmes le reconnaissent :

> *« Vérifier les dépendances entre custom nodes et autres custom nodes est
> pratiquement impossible. »* — docs ComfyUI

Symptôme concret : **ComfyUI-Manager a dû abandonner** `pip install -r
requirements.txt` au profit d'un `pip install <package>` **ligne par ligne**,
car avec ~100 nœuds une résolution groupée « échouait avec forte probabilité ».
Conséquence : **dernier qui écrit gagne**, sans résolution croisée. Le
`requirements.txt` du nœud B peut **désinstaller silencieusement** la version
dont le nœud A a besoin.

**Conflits réels, vérifiés sur GitHub :**

- **NumPy 1.x vs 2.x** (le plus cité) — une maj ComfyUI tire `numpy>=2`, et les
  modules compilés contre NumPy 1.x crashent : *« A module compiled using NumPy
  1.x cannot be run in NumPy 2.x »*. C'est une incompatibilité **ABI**, pas un
  simple numéro de version. Fix : épingler `numpy<2`. (issues
  Comfy-Org/ComfyUI#9156, ComfyUI-Manager#2053)
- **OpenCV ↔ NumPy** : `opencv-python 4.12` exige `numpy<2.3,>=2`, ce qui entre
  en collision avec les nœuds qui épinglent `numpy<2`.
- **`huggingface_hub`** : versions divergentes selon les nœuds → classes qui ne
  chargent plus, workflows inutilisables.

### 1.2 Le cauchemar torch / torchvision / CUDA / xformers

Le pire offenseur, car c'est une **matrice couplée et spécifique au build
CUDA** : `torch`, `torchvision`, `torchaudio` doivent être la **même version**
buildée pour le **même tag CUDA** (`+cu121`, `+cu126`…), servie depuis l'index
PyTorch — et `xformers` est compilé contre **une** version torch exacte.

**Exemple vérifié (issue #8882) :** un utilisateur a `torch==2.5.1+cu121`.
Installer `xformers 0.0.31.post1` (qui déclare `torch==2.7.1`) provoque :

```
Collecting torch==2.7.1 (from xformers)
  → désinstalle 2.5.1+cu121, installe 2.7.1 (wheel CPU depuis PyPI, PAS le build CUDA)
ERROR: torchaudio 2.5.1+cu121 requires torch==2.5.1+cu121, but you have torch 2.7.1
```

Résultat : un torch **CPU-only cassé** là où le GPU marchait. **Un simple nœud
qui liste `xformers` (non épinglé) peut arracher tout le build CUDA d'une
install fonctionnelle.**

### 1.3 Le trou « works on my machine »

Le **workflow `.json` n'enregistre que le graphe** (nœuds, connexions,
paramètres) — **pas** les versions des custom nodes, ni les paquets pip, ni le
build torch/CUDA, ni les fichiers de modèle. ComfyUI récent ajoute `cnr_id` et
`ver` sur les nœuds pour *aider au diagnostic*, mais ça ne capture que
**l'identité du nœud**, pas le graphe de dépendances pip résolu. **Deux
personnes chargeant le même JSON obtiennent des versions différentes et des
résultats différents.** C'est précisément ce trou qu'un lockfile doit fermer.

---

## 2. Comment ComfyGit s'y attaque

[`comfygit-ai/comfygit`](https://github.com/comfygit-ai/comfygit) — **Python,
GPL-3.0, v0.5.0** (« MVP / early release », ~19 ★). C'est du **vrai `git` +
vrai `uv`** (sous-process réels), pas une métaphore. Le code est nettement plus
mûr que ne le laissent croire les étoiles, mais **non éprouvé** en production.

### 2.1 Le modèle : environnements isolés + recette portable

- **Granularité = par environnement (≈ par projet), PAS par workflow.** Chaque
  environnement a son `pyproject.toml`, son `.venv/`, son checkout `ComfyUI/`.
  Les workflows vivent *dans* l'env. Deux workflows aux deps incompatibles ? On
  les **sépare en deux environnements**.
- **Modèles partagés par symlink** : un répertoire `models/` global, symlinké
  dans chaque env → N environnements = **1 seule copie** du checkpoint de 6 Go.
- **Ce qui est versionné dans git** : le `pyproject.toml` (le manifeste), les
  `workflows/`, les `workflow_api/`, les **métadonnées** de nœuds/modèles + les
  **URLs sources** des modèles. **Pas** versionné (gitignoré) : les octets des
  modèles, `.venv/`, le checkout `ComfyUI/`, `uv.lock`, `.pytorch-backend`.

### 2.2 Comment il dompte torch/CUDA (la partie qui nous intéresse le plus)

- Le backend torch est stocké dans un fichier **gitignoré** `.pytorch-backend`
  (ex. `cu128`, `cpu`, `rocm6.3`, `xpu`) → l'index est dérivé en
  `https://download.pytorch.org/whl/{backend}`.
- **Décision de design clé :** le backend torch est **délibérément local à la
  machine, jamais commité**. Le même commit d'env tourne ainsi sur une machine
  CUDA *et* sur un Mac. Override par opération : `cg sync --torch-backend cu128`.

### 2.3 Comment il dompte les `requirements.txt` concurrents

- À `cg node add`, les requirements du nœud sont rangés dans un
  **`[dependency-groups]` dédié** du `pyproject.toml`, au nom **anti-collision**
  `{nom-normalisé}-{sha256(repo_url)[:8]}`.
- Avant de committer, il **teste les deps en isolation** (résolution uv sur une
  copie temporaire du pyproject). `--strict` échoue plutôt que d'auto-résoudre.
- Conflits → `cg constraint add "numpy<2"` (portable), `cg overlay` (override
  local), ou séparation d'environnements.

> ⚠️ **Tous les nœuds partagent quand même UN venv par environnement.**
> L'isolation entre nœuds est par *dependency-group uv* (métadonnées), pas par
> venvs séparés. Des stacks réellement incompatibles ne peuvent pas coexister.

### 2.4 La boucle de réconciliation (cruciale pour nous)

Ce **n'est pas** qu'un export/import de snapshot. Le modèle est **manifeste
portable → runtime dérivé**, avec réconciliation explicite :

- `cg sync` — réconcilie le runtime pour matcher le manifeste (résout/installe
  via uv, applique le backend torch, installe les groupes de nœuds, recrée le
  venv si besoin, reconstruit les symlinks de modèles). **C'est la boucle apply.**
- `cg repair` — réconcilie quand le runtime a dérivé / est endommagé.
- `cg run` **sync d'abord par défaut** ; pull/checkout/switch/merge/revert
  **auto-réconcilient** ensuite.
- `cg materialize <git-url|tar|dir>` — **le chemin headless/CI** : pas de
  prompts, manager sauté sauf `--with-manager`, modèles sautés sauf demande, et
  **un échec de sync fait échouer la commande**. C'est le point d'entrée le plus
  pertinent pour un orchestrateur.

> **Important :** la réconciliation est **déclenchée par commande**
> (`sync`/`repair`/`materialize`), **pas un contrôleur continu**. Aucun daemon
> ne surveille le remote git pour converger tout seul. **L'orchestrateur devra
> piloter la boucle.**

---

## 3. Le paysage des solutions (comparatif)

| Outil | Isolation | Ce qui est épinglé | torch/CUDA | Réconciliation | Lockfile pip résolu |
|-------|-----------|--------------------|------------|----------------|---------------------|
| **ComfyUI-Manager** snapshot | 1 venv partagé | core commit + nœuds (git hash/CNR ver) + **`pip list` brut** | non distinct | impératif (save/restore) | non (état brut, pas résolu) |
| **comfy-cli** `comfy-lock.yaml` | 1 venv | core commit + nœuds + modèles (url+hash+type) | **pas dans le schéma** (dataclass `CustomNode` = stub `pass`) | impératif | non (schéma) |
| **comfy-cli** `DependencyCompiler` (`uv.py`) | 1 venv | — | **OUI** : `make_override()` épingle torch/vision/audio + `extra_index_url` par GPU | install-time | **OUI** : `uv pip compile` → `requirements.compiled` (mais **non persisté** dans le lockfile !) |
| **comfy-pack** `.cpack.zip` | venv neuf rebuildé | **paquets pip + nœuds@commit + modèles@hash** | (via versions épinglées) | snapshot/pack (impératif) | proche (versions épinglées dans le zip) |
| **comfy-env** (pixi) | **par nœud** (subprocess workers + sockets) | conda+pip+wheels CUDA par nœud, vrai lockfile pixi | oui (wheels CUDA prébuildés) | impératif | oui (pixi) |
| **ComfyGit** | **par environnement** (1 venv/env) | nœuds@commit + deps (groups) + modèles (url+hash) | **oui** (backend local, jamais commité) | **OUI, boucle `sync`/`materialize`** | **non** (`uv.lock` gitignoré) |
| **Orchestrateur visé** | par env (réutilise ComfyGit) | tout, déclaratif dans git | oui | **continue + drift** | **oui (objectif)** |

**Deux enseignements forts :**

1. **comfy-cli a déjà le meilleur *résolveur* torch-aware** (`DependencyCompiler`
   + `uv pip compile` + override GPU) — mais il **ne persiste pas** le
   `requirements.compiled` résolu dans `comfy-lock.yaml`. Il y a un **trou entre
   le bon résolveur et le lockfile faible**.
2. **ComfyGit a déjà la meilleure *boucle de réconciliation* + isolation par
   env + gestion torch locale** — mais **`uv.lock` est gitignoré**, donc la
   reproductibilité est « recette », pas « bit-à-bit ».

> Personne ne combine : *résolveur uv torch-aware* + *lockfile résolu persisté*
> + *isolation par env* + *réconciliation continue avec drift*. **C'est l'union
> manquante que l'orchestrateur peut viser.**

---

## 4. Ce que ComfyGit ne fait PAS (les trous)

Ce sont exactement les espaces que l'orchestrateur doit combler :

1. **Pas de lockfile commité** → reproductibilité « recette », pas « bit-à-bit ».
   `uv.lock` est gitignoré (« machine-specific à cause des variantes torch »).
   Deux machines sur le même commit peuvent résoudre des transitives différentes.
2. **Pas de contrôleur continu.** `sync`/`repair`/`materialize` convergent
   *quand on les invoque*. Aucun daemon ne surveille git. → **L'orchestrateur
   est ce contrôleur.**
3. **Pas de gestion de parc / d'instances distantes.** Collaboration = remotes
   git + tarballs. `cg serve` ne fait que fronter un ComfyUI **local** ;
   `materialize` **ne lance pas** ComfyUI ni ne bind de port. → **L'orchestrateur
   gère le multi-instance / remote.**
4. **« Content-addressable » survendu** : répertoire partagé + symlinks + index
   SQLite sur un **xxhash échantillonné** (pas un CAS Merkle). L'acquisition
   cross-machine dépend d'URLs sources ajoutées à la main.
5. **Isolation limitée à 1 venv/env** : des nœuds réellement incompatibles
   forcent la séparation d'environnements (vs `comfy-env` qui isole par nœud).
6. **Maturité** : v0.5, ~19★, API « peut changer », **GPL-3.0** (copyleft → à
   surveiller si on *embarque/lie* `comfygit-core` dans un produit distribué ;
   l'**invoquer en sous-process via le CLI `cg` n'est pas du linking**).

---

## 5. Lien avec l'orchestrateur : où ComfyGit s'insère

Rappel de l'archi de l'orchestrateur (cf. doc précédent) : un
`comfyui.lock.yaml` versionné = état désiré ; un `core/reconciler.py` qui diffe
désiré ↔ réel ; `apply` / `status` / `diff` ; et un cœur de valeur =
**réconciliation de dépendances + détection de drift**.

**ComfyGit recouvre quasiment toute la couche « install + dépendances + env »
de cet orchestrateur.** La cartographie :

| Besoin de l'orchestrateur | Fourni par ComfyGit ? | Détail |
|---------------------------|-----------------------|--------|
| Créer un env ComfyUI reproductible | ✅ | `cg create` / `cg materialize` (uv + torch backend + nœuds@commit) |
| Installer nœuds @SHA | ✅ | `cg node add`, dependency-groups, test d'isolation |
| Gérer torch/CUDA proprement | ✅ | backend local, override par op |
| Symlinks modèles (dédup) | ✅ | répertoire partagé + index |
| Boucle apply (converge runtime→manifeste) | ✅ (déclenchée) | `cg sync` / `cg materialize` |
| **Réconciliation CONTINUE (watch git → apply)** | ❌ | **→ orchestrateur** |
| **Détection de drift (`status`/`diff`)** | ⚠️ partiel (`cg status`/`repair`) | drift **runtime** seulement ; pas de diff modèle par hash de premier rang → **orchestrateur** |
| **Lockfile pip résolu, commité, bit-à-bit** | ❌ (`uv.lock` gitignoré) | **→ orchestrateur** (ou patcher : committer `uv.lock` + pin du backend) |
| **Multi-instance / parc distant** | ❌ | **→ orchestrateur** |
| Exécution de workflow + download sorties | ❌ (hors scope) | **→ orchestrateur** (`POST /prompt` + WS + `/view`) |

> **Conclusion :** ComfyGit est un **excellent candidat comme « couche moteur
> d'environnement »** de l'orchestrateur. L'orchestrateur ne réimplémente PAS la
> gestion uv/torch/nœuds — il **pilote `cg materialize`/`cg sync`** et ajoute par
> dessus les 4 briques manquantes : **(1) contrôleur continu, (2) drift de
> modèles par hash, (3) lockfile résolu commité, (4) parc multi-instance +
> exécution.**

---

## 6. Trois scénarios d'intégration

### Scénario A — « Orchestrateur sur ComfyGit » (build vs buy)
L'orchestrateur **enveloppe le CLI `cg`** en sous-process. Le `comfyui.lock.yaml`
de l'orchestrateur référence/encapsule un environnement ComfyGit (un repo git
par env). `apply` = `git pull` + `cg materialize/sync`. `status` = `cg status`
+ notre diff de hash de modèles.
- ✅ Le plus rapide ; on hérite de la gestion uv/torch/nœuds, déjà mûre.
- ⚠️ Dépendance à un projet v0.5 (API instable) ; GPL-3.0 (OK en sous-process,
  pas en linking) ; `uv.lock` gitignoré à contourner.
- 👉 **Recommandé pour le MVP** : valider la valeur (drift + réconciliation
  continue + exécution) sans réécrire la couche env.

### Scénario B — « Inspiré de, mais autonome »
On **réimplémente** la couche env en s'inspirant de ComfyGit *et* du
`DependencyCompiler` de comfy-cli (le meilleur résolveur torch-aware), mais on
**persiste le `requirements.compiled` résolu** dans notre lockfile (ce qu'aucun
ne fait).
- ✅ Indépendant, lockfile bit-à-bit, licence libre (MIT/Apache).
- ⚠️ Beaucoup plus de travail ; on refait la gestion torch/CUDA (la partie dure).

### Scénario C — « Hybride » (probable cible long terme)
MVP en **A** (wrapper ComfyGit), puis on **internalise** progressivement les
parties critiques (lockfile résolu commité, drift de modèles) tout en gardant
ComfyGit comme backend par défaut et en supportant `comfy-cli`/`comfy-pack`
comme backends alternatifs via le **pattern providers** (cf. `agents/` de
comfy-pilot).

---

## 7. Recommandation

1. **Ne pas réimplémenter la gestion uv/torch/nœuds.** C'est la partie dure et
   ComfyGit (+ le `DependencyCompiler` de comfy-cli) la font déjà bien.
2. **MVP = Scénario A** : orchestrateur qui pilote `cg materialize`/`cg sync`,
   et qui **ajoute la valeur manquante** :
   - **détection de drift de modèles par hash** (le mismatch silencieux qui
     corrompt les sorties sans erreur),
   - **réconciliation continue** (watch d'un remote git → `apply`),
   - **exécution de workflow** (`POST /prompt` + WS + download).
3. **Contourner le trou du lockfile** dès le MVP : à `apply`, **committer
   `uv.lock` + le backend torch résolu** dans *notre* état désiré (pas dans le
   repo ComfyGit), pour obtenir la repro bit-à-bit que ComfyGit refuse par choix.
4. **Abstraire le backend d'env derrière une interface** (`providers/env/`)
   dès le départ → ComfyGit aujourd'hui, comfy-cli/comfy-pack/comfy-env demain,
   migration vers une implémentation maison (Scénario B) sans casser le CLI.
5. **Licence** : garder l'orchestrateur en MIT/Apache, n'invoquer ComfyGit
   **que via son CLI** (`cg`), jamais par import de `comfygit-core` (GPL).

---

## 8. Sources

**ComfyGit (vérifié sur source, commit `2e336bf`, 2026-05-27)**
- Repo / README : <https://github.com/comfygit-ai/comfygit>
- Sync/repair (boucle de réconciliation), materialize (headless/CI) : `docs/comfygit-docs/docs/user-guide/...`
- Backend torch : `packages/core/src/comfygit_core/managers/pytorch_backend_manager.py`, `utils/pytorch.py`
- Dependency-groups par nœud : `packages/core/src/comfygit_core/manifest/nodes.py`
- Hash/index modèles : `repositories/model_repository.py` ; symlinks : `managers/model_symlink_manager.py`
- `uv.lock` gitignoré : `managers/git_manager.py`

**Enfer des dépendances + outils**
- ComfyUI-Manager (install ligne par ligne, snapshot, `pip list`) : <https://github.com/Comfy-Org/ComfyUI-Manager/blob/main/glob/manager_core.py>, `glob/manager_util.py`, `docs/en/cm-cli.md`
- comfy-cli `comfy-lock.yaml` (schéma + dataclass stub) : <https://github.com/Comfy-Org/comfy-cli/blob/main/README.md>, `comfy_cli/workspace_manager.py`
- comfy-cli `DependencyCompiler` (uv pip compile + override torch) : <https://github.com/Comfy-Org/comfy-cli/blob/main/comfy_cli/uv.py>
- comfy-pack (`.cpack.zip`) : <https://github.com/bentoml/comfy-pack>
- comfy-env (isolation par nœud, pixi) : <https://github.com/PozzettiAndrea/comfy-env>

**Conflits réels (issues GitHub)**
- NumPy 1.x/2.x : <https://github.com/Comfy-Org/ComfyUI/issues/9156>, <https://github.com/Comfy-Org/ComfyUI-Manager/issues/2053>
- xformers → torch 2.7.1 (CUDA cassé) : <https://github.com/Comfy-Org/ComfyUI/issues/8882>
- xformers C++/CUDA mismatch : <https://github.com/comfyanonymous/ComfyUI/issues/5246>
- « Custom Nodes as Python Dependencies » : <https://github.com/Comfy-Org/ComfyUI/discussions/1959>
</content>
