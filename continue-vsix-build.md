# Compiler Continue (VS Code) avec le patch — VSIX win32-x64

Instructions pour compiler l'extension VS Code **Continue v2.1.0** avec le patch
[`continue-improvements.patch`](./continue-improvements.patch) (14 commits), sur **Windows** avec **Node 24**.

## Patch optionnel n°2 

[`continue-cli-modes.patch`](./continue-cli-modes.patch) (**4 commits**) s'applique **après** le premier patch
(`git am continue-improvements.patch` puis `git am continue-cli-modes.patch`) et ajoute au **CLI** :

- **Mode chat** (sans tools, conversation simple) — mode de permission `chat`, dans le cycle Shift+Tab
  et via `--mode chat` :
  ```powershell
  cn --mode chat          # conversation pure, aucun outil offert au modèle
  cn --mode plan          # lecture seule (existant)
  cn --mode normal        # agent avec approbations (existant)
  cn --mode auto          # tout permis (existant)
  ```
- **Mode tools `text`** pour serveurs OpenAI-compatibles au function calling peu fiable :
  les outils sont décrits dans le system message et les appels sont parsés depuis des blocs
  ```` ```tool ```` de la réponse — le paramètre natif `tools` n'est **jamais** envoyé.
  ```powershell
  cn --tool-mode text                 # au démarrage
  cn                                  # puis dans l'interface :
  #   /tool-mode          -> bascule native <-> text
  #   /tool-mode text     -> force text
  #   Shift+Tab           -> cycle normal/plan/auto/chat
  #   l'indicateur affiche [⏸ plan · tools:text] etc.
  ```
- **Résilience mid-stream** (inspirée des mid-stream retries de Gemini CLI) :
  - 429/5xx/connexions coupées → jusqu'à 3 retries (backoff 1s/2s/4s)
  - **continuation** : réponse partielle renvoyée en prefill assistant → le modèle **continue**
    sans régénérer ni dupliquer ; stream thinking-only/vide → retry avec un nudge
    « provide your final answer » ; une fois le budget épuisé, l'erreur s'affiche
  - **watchdog d'inactivité** par chunk (120 s par défaut, `CONTINUE_CLI_IDLE_TIMEOUT_MS`, 0 = off) :
    un serveur surchargé qui se tait est traité comme une panne retryable
- **Streaming live** dans le TUI : le contenu en cours s'affiche par lots (~120 ms)
  au lieu d'apparaître seulement à la fin
- **Token estimation hybride** : les textes > 20k chars utilisent une estimation
  (fini les freezes de l'UI pendant la validation de contexte)
- **Télémétrie strictement opt-in** : `CONTINUE_CLI_ENABLE_TELEMETRY=1` requis
- **GUI VS Code** (commit 4 du patch) : `StepContainer` (rendu markdown des messages
  assistant) **mémoïsé** → pendant un stream, seul le message en cours se re-rend
  (avant : toute l'historique visible ~8-20×/s) ; poll du stream webview 50 ms → **120 ms**
- Combinable : `cn --mode plan --tool-mode text` ; en mode chat, aucun outil n'est proposé
  (text ou natif) et le system message est dédié à la conversation.
- Validation : tsc CLI ✅, **480+ tests vitest CLI ✓** (dont 4 nouveaux sur la résilience :
  continuation sans duplication, nudge thinking-only, épuisement des retries,
  pas de double-reprise connection-phase), GUI `tsc` + tests stream **16/16 ✓**,
  empilement `git am` patch1+patch2 vérifié sur `main`.

> ⚠️ Le VSIX `win32-x64` doit être compilé **depuis Windows** : les modules natifs
> (LanceDB, sqlite, ripgrep) sont copiés pour la plateforme de build. La CI officielle
> compile chaque cible sur son propre runner. Ne cross-compilez pas depuis Linux —
> `--target` ne change que les métadonnées, pas les binaires.

---

## 1. Prérequis

| Outil | Version |
|---|---|
| Node.js | **v24.x** (validé : v24.22.3 / npm 11.x ; `engines` du repo : `node >= 20.20.1`) |
| git | récent (pour `git am`) |
| VS Code | pour tester/installer le VSIX |

## 2. Récupérer le code + le patch

```powershell
git clone https://github.com/continuedev/continue continue
cd continue
git checkout v2.1.0-vscode        # ou main (état final public, repo archivé)

# Le patch est dans ce repo :
# https://github.com/samskinov/csharp-examples/blob/main/continue-improvements.patch
curl.exe -LO https://raw.githubusercontent.com/samskinov/csharp-examples/main/continue-improvements.patch

git am continue-improvements.patch        # 14 commits
# (secours si git am échoue : git apply -3 continue-improvements.patch)
```

## 3. Contenu du patch (14 commits)

| # | Commit | Ce que ça corrige |
|---|---|---|
| 1 | `fix(llm)` | check `response.ok` + erreur typée avec `.status` (chat, responses, fim) |
| 2 | `fix(gui)` | retry 429/rate-limit/quota/coupures (4 essais, backoff 2→16 s) **sans doublon de message** ; le partiel conservé sert de prefill → la réponse **continue** au lieu de re-thinker |
| 3 | `fix(llm)` | ne plus perdre le `reasoning_content` d'un delta portant content + reasoning |
| 4 | `perf` | suppression des appels bloquants (`execSync` esbuild config.ts, SecretStorage en fs sync) |
| 5 | `fix(tools)` | guidage LLM : `read_file_range`, hiérarchie edit vs find&replace |
| 6 | `feat(llm)` | note visible quand `finish_reason=length` (thinking qui épuise le budget sans réponse) |
| 7 | `feat(gui)` | **watchdog d'inactivité** (120 s par défaut, `experimental.streamIdleTimeoutMs`, 0 = off) + reprise auto du tour |
| 8 | `feat(llm)` | `reasoningBudgetTokens` → `reasoning_effort` (o-series/gpt-5 uniquement) |
| 9 | `fix(llm)` | log des chunks Responses malformés (fin des erreurs silencieuses) |
| 10 | `fix(gui)` | message « Premature Close » à jour (mentionne le retry auto) |
| 11 | `perf(llm)` | **tokenisation du contexte via le pool de workers** (`llm/compileChat` était sync en JS pur → cause n°1 du freeze à l'envoi) |
| 12 | `perf(llm)` | tokenisation du prompt/réponse/**thinking** en fin de stream via workers (cause n°2) |
| 13 | `perf(history)` | écriture de session **asynchrone** (JSON multi-Mo + `writeFileSync` sync → cause n°3 du freeze en fin de tour) |
| 14 | `fix(llm)` | chemin du tokenizer worker compatible jest **et** vitest |

## 4. Compiler le VSIX (PowerShell, cible `win32-x64`)

```powershell
# 0) Node 24 dans le PATH
node --version    # attendu : v24.x

# 1) Packages internes + core — l'ordre de build est géré par le script du repo
node .\scripts\build-packages.js

# 2) Dépendances
cd extensions\vscode ; npm ci ; cd ..\..
cd gui ; npm ci ; cd ..

# 3) Webview (GUI React)
cd gui
npm run build        # NODE_OPTIONS="--max-old-space-size=4096" conseillé
cd ..

# 4) Prépackage + VSIX
cd extensions\vscode
$env:GITHUB_TOKEN = "<votre-token-github>"   # téléchargements vscode-ripgrep / sqlite
npm run prepackage -- --target win32-x64
npm run package -- --target win32-x64
#   → extensions\vscode\build\continue-<version>.vsix
```

Installation :

```powershell
code --install-extension extensions\vscode\build\continue-<version>.vsix
```

### Variante linux-x64 (bash)

```bash
node ./scripts/build-packages.js
cd extensions/vscode && npm ci && cd ../..
cd gui && npm ci && npm run build && cd ..
cd extensions/vscode
GITHUB_TOKEN="<token>" npm run prepackage -- --target linux-x64
GITHUB_TOKEN="<token>" npm run package -- --target linux-x64
#   → extensions/vscode/build/continue-<version>.vsix
```

## 5. Notes de build

- **`GITHUB_TOKEN`** : requis par `@vscode/ripgrep` et le téléchargement des binaires
  sqlite (rate-limit GitHub) — c'est ce que la CI passe à ces étapes.
- **`SKIP_INSTALLS=true`** avant `prepackage` : saute les réinstallations lourdes quand
  `node_modules` sont déjà en place (itérations rapides).
- Si `npm ci` de `core` réclame `vectordb` (étape présente en CI) : `cd core ; npm i vectordb`
  — pas nécessaire dans l'environnement de validation de l'audit.
- La chaîne de build des packages internes est **ordonnée obligatoirement** :
  `config-types → terminal-security → llm-info → config-yaml → fetch → openai-adapters → core → gui → extensions/vscode`.
  `scripts/build-packages.js` s'en charge ; ne les construisez pas à la main dans le désordre
  (`Cannot find module '@continuedev/...'`).
- Piège git : le repo Continue contient un **tag nommé `main`** qui rend `main..branche`
  ambigu — utiliser `refs/heads/main..` dans les comparaisons.
- npm 11 réécrit `core/package-lock.json` pendant l'install : ne pas le commiter.

## 6. Proxy d'entreprise (407) — symptômes réels et parades

Erreur **407** = le proxy d'entreprise exige une authentification. Chaque
téléchargeur a sa propre variable de contournement — les quatre premiers
ont tous été rencontrés sur un poste corporate Windows.

### 6.1 Tableau des téléchargements → parades

| Téléchargement | Source / format | Parade |
|---|---|---|
| **headers Node** (node-gyp compile `sqlite3`, etc.) | nodejs.org, `.tar.gz` | `npm config set nodedir "C:\Program Files\nodejs"` (utilise les headers du MSI local, **zéro download**) ou `$env:NODEJS_ORG_MIRROR` vers un miroir |
| **libvips** (via `@xenova/transformers` → `sharp@0.32.6` de `core`) | GitHub `sharp-libvips`, **`.tar.br`** (Brotli) | télécharger le `.tar.br` dans le navigateur, le servir localement, puis `$env:SHARP_DIST_BASE_URL = "http://127.0.0.1:8080/"` |
| **binaires sharp 0.33** (`@img/sharp-win32-x64`) | registry npm (optionalDependencies) | `npm i @img/sharp-win32-x64@0.33.5 --save-optional` — les échecs d'optional deps sont **silencieux**, le problème ne se voit qu'au 1er `import("sharp")` |
| **ripgrep** (`@vscode/ripgrep`, postinstall) + **binaires sqlite** (prepackage VSIX) | GitHub releases | proxy npm authentifié + `$env:GITHUB_TOKEN` |
| **registry npm** (tout le reste) | registry.npmjs.org | `npm config set proxy/https-proxy "http://USER:PASS@proxy:8080"` |

### 6.2 Les quatre recettes rencontrées

**a) 407 sur les headers Node (build VSIX)**
```powershell
$env:npm_config_nodedir = "C:\Program Files\nodejs"   # session
npm config set nodedir "C:\Program Files\nodejs"       # persistant
# variante miroir : $env:NODEJS_ORG_MIRROR = "http://mirror.corp/nodejs-dist"
```

**b) sharp 0.33 : « Could not load sharp » + 407 sur fallback libvips**
```powershell
npm config get omit          # doit être vide (sinon optional deps désactivées)
cd extensions\cli
npm i @img/sharp-win32-x64@0.33.5 --save-optional
node -e "import('sharp').then(()=>console.log('sharp OK'))"
# fallback 100% manuel : télécharger le .tgz via le navigateur puis
#   npm i .\sharp-win32-x64-0.33.5.tgz --save-optional
```

**c) 407 sur un `.tar.br` (libvips de `sharp@0.32.6`, tiré par
`@xenova/transformers` de core)**

Le downloader (`simple-get`) **ignore le proxy npm** → miroir local obligatoire.
Version attendue par sharp 0.32.6 : **libvips 8.14.5** (le nom exact figure dans
le message d'erreur).

Recette **sans serveur HTTP** (option du code sharp : `sharp_libvips_local_prebuilds`,
cherche `<dossier>\v<version>\<tarball>`) :
```powershell
# 1) navigateur : https://github.com/lovell/sharp-libvips/releases/download/v8.14.5/libvips-8.14.5-win32-x64.tar.br
# 2) layout exigé : C:\sharp-dist\v8.14.5\libvips-8.14.5-win32-x64.tar.br
npm config set sharp_libvips_local_prebuilds "C:\sharp-dist"
# 3) relancer l'install de core - le log doit dire "Using local libvips from ..."
```

Variante serveur HTTP (attention aux pièges) :
```powershell
python -m http.server 8080 --directory C:\sharp-dist
$env:SHARP_DIST_BASE_URL = "http://127.0.0.1:8080/"   # slash final OBLIGATOIRE
# fichier À PLAT dans le dossier servi (pas de sous-dossier v8.14.5 avec cette option)
```

**Pièges** :
- `npm_config_sharp_dist_base_url` (dans .npmrc) **écrase** `SHARP_DIST_BASE_URL`
  → vérifier `npm config get sharp_dist_base_url` ;
- env `$env:` définie dans un autre terminal que celui d'npm ;
- nom du tarball différent de celui demandé (se fier au message d'erreur) ;
- **simple-get n'utilise pas le proxy npm** → seul le miroir/local fonctionne pour
  ce download (c'est pourquoi le 407 persistait malgré npm config proxy).

**d) `comment-json` introuvable au build du CLI**
- `npm ls comment-json` doit être lancé **dans `core/`** (dep de core, pas du
  CLI ni de la racine — « not found » ailleurs est normal)
- introuvable même dans `core/` → arbre incomplet (install avortée par un 407) :
```powershell
cd core
If (Test-Path node_modules) { Remove-Item -Recurse -Force node_modules }
npm ci ; npm run build
cd ..\extensions\cli ; npm run build ; node dist\cn.js --help
```

### 6.3 npm ≥ 9/11 rejette `npm config set` pour ces clés (« not a valid option »)

npm valide désormais les clés : les options tierces (sharp, etc.) se posent
**directement dans `.npmrc`** (npm passe tous les champs en `npm_config_*` aux
scripts d'install) :

```ini
# %USERPROFILE%\.npmrc  (global) ou core\.npmrc (projet-local)
sharp_libvips_local_prebuilds=C:/sharp-dist
```

Ou par **variables d'environnement** (le nom = `npm_config_` + option en
minuscules) :

```cmd
:: CMD - session courante
set npm_config_sharp_libvips_local_prebuilds=C:\sharp-dist
:: CMD - persistant (nouvelle fenêtre requise ensuite)
setx npm_config_sharp_libvips_local_prebuilds "C:\sharp-dist"
```
```powershell
# PowerShell - session courante
$env:npm_config_sharp_libvips_local_prebuilds = "C:\sharp-dist"
```

Vérifier dans le **même** terminal : `echo %npm_config_sharp_libvips_local_prebuilds%`
(CMD) / `$env:npm_config_sharp_libvips_local_prebuilds` (PS), puis le log
d'install doit dire « Using local libvips from … ».

### 6.4 Proxy NTLM / Kerberos (le 407 persiste avec identifiants)

npm et node-gyp ne parlent pas NTLM. Installer un proxy local qui gère
l'authentification Windows — **Px** ou **Cntlm** — puis tout pointer dessus :

```powershell
npm config set proxy "http://127.0.0.1:3128"
npm config set https-proxy "http://127.0.0.1:3128"
```

Divers :
- mot de passe dans l'URL : **URL-encoder** les caractères spéciaux (`@`→`%40`…) ;
- TLS inspecté : `$env:NODE_EXTRA_CA_CERTS = "C:\certs\corp-ca.pem"` ;
- mettre les env **avant** `build:local-deps` (le `npm install` de `core` est la
  première compilation : sqlite3 y est gypé) ;
- diagnostic : chaque message d'erreur 407 mentionne son domaine — un domaine
  non couvert = une env à ajouter.

## 7. Compiler le CLI (`cn`)

Le CLI vit dans `extensions/cli` et se bundle en un fichier autonome (esbuild,
12,7 Mo, `dist/cn.js`) — les packages internes et `core` sont embarqués depuis
la source (alias dans `build.mjs`).

```bash
export PATH=<node24>/bin:$PATH      # Node 24 validé
cd <votre-clone-de-continue>/extensions/cli

# 1) Dépendances + build des packages internes et de core (ordre géré par le script)
npm run build:local-deps            # (~5-10 min sur un clone frais)

# 2) Build du bundle (validate aliases + esbuild)
npm run build                       # → dist/index.js + dist/cn.js (wrapper)

# 3) Utilisation
node dist/cn.js --version           # sanity check (exige les patchs : les flags
node dist/cn.js --help              #   --mode / --tool-mode apparaissent dans --help)
node dist/cn.js --mode chat
node dist/cn.js --mode plan --tool-mode text

# 4) (optionnel) commande `cn` globale pour la session courante
npm link                            # crée le binaire `cn` pointant sur dist/cn.js
```

Notes :
- `npm run build:local-deps` n'est nécessaire **qu'une fois** par clone (les `dist/`
  des packages et de `core` persistent) ; ensuite `npm run build` suffit.
- Sur Windows : mêmes commandes en PowerShell (`npm.cmd` via npx automatique) ;
  pas de binaire natif à compiler (fsevents macOS et le worker JSDOM sont gérés
  par le script).
- Version affichée `0.0.0-dev` en local : la version définitive vient du
  `semantic-release` de la CI.
- Logs : `npm run watch:logs` (tail de `~/.continue/logs/cn.log`).

## 8. Statut de validation

- Chaîne complète exécutée avec **node v24.22.3 / npm 11.19.1** : packages → core (`tsc`) →
  gui (`tsc` + `vite build`) → extensions/vscode (`esbuild`, bundle `out/extension.js`
  généré avec le patch complet).
- Tests : vitest OpenAI **304/304**, jest `openaiTypeConverters`+`countTokens` **63 ✓**,
  jest `util/history` **18/18**, tests GUI stream **16/16** (patch 1) ;
  **480+ tests CLI ✓** dont 4 sur la résilience mid-stream, tests GUI stream **16/16** (patch 2).
- `git am` vérifié sur `main` propre : **14/14 commits** (patch 1),
  puis **4/4** en empilement (patch 2) — **18 commits** au total.
- Le pas final `vsce package` est celui de la CI officielle
  (`.github/actions/build-vscode-extension/action.yml`) ; à exécuter sur votre machine.

> ⚠️ **Ordre et fichiers** : deux patchs uniquement, dans cet ordre —
> 1. `continue-improvements.patch`, 2. `continue-cli-modes.patch`.
> Un ancien fichier `continue-cli-chatmode.patch` (rename du patch 2 dans sa
> version 1-commit) a été **retiré du repo** : son contenu est inclus dans
> `continue-cli-modes.patch` — l'appliquer en plus échouerait en conflits.
