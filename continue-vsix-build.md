# Compiler Continue (VS Code) avec le patch — VSIX win32-x64

Instructions pour compiler l'extension VS Code **Continue v2.1.0** avec le patch
[`continue-improvements.patch`](./continue-improvements.patch) (14 commits), sur **Windows** avec **Node 24**.

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

## 6. Statut de validation

- Chaîne complète exécutée avec **node v24.22.3 / npm 11.19.1** : packages → core (`tsc`) →
  gui (`tsc` + `vite build`) → extensions/vscode (`esbuild`, bundle `out/extension.js`
  généré avec le patch complet).
- Tests : vitest OpenAI **304/304**, jest `openaiTypeConverters`+`countTokens` **63 ✓**,
  jest `util/history` **18/18**, tests GUI stream **16/16**.
- `git am` vérifié sur `main` propre : **14/14 commits**.
- Le pas final `vsce package` est celui de la CI officielle
  (`.github/actions/build-vscode-extension/action.yml`) ; à exécuter sur votre machine.
