# Installer / compiler Continue (VS Code + CLI) avec les patchs

Ce repo fournit **4 patchs** à appliquer sur `continuedev/continue` (v2.1.0 ou
main) qui suppriment les dépendances aux binaires natifs (SQLite natif,
ripgrep) et rendent le build reproductible depuis le registry npm seul — pensés
pour les postes derrière un proxy d'entreprise.

## Les 4 patchs (dans cet ordre)

| Ordre | Fichier | Contenu |
|---|---|---|
| 1 | `continue-improvements.patch` | 14 commits : résilience stream/thinking/429, freezes (tokenisation workers, sessions async), tools, watchdog |
| 2 | `continue-cli-modes.patch` | 4 commits : CLI chat/plan/agent + tools text, streaming live, perf GUI |
| 3 | `continue-fallbacks.patch` | 1 commit : SQLite WASM (plus de binding natif), `dbinfoz`/`@database` retiré, recherche intégrée (plus de ripgrep), prepackage adapté |
| 4 | `continue-cli-tools.patch` | 6 commits : chat sans tools natifs, un seul system message en tête, `/tool-mode` visible, réparation des tool calls XML (`</arg_value>`, `file_path`) côté CLI et VS Code, fallback XML en mode texte, VSIX limité au binaire LanceDB de la cible |

⚠️ Vérification faite : empilement `git am` 14+4+1+6 = **25 commits** sur `main`
propre. Version de référence : `v2.1.0-vscode` (tag) ou `main` (état final).

## 1. Clone + patchs

```cmd
git clone https://github.com/continuedev/continue continue
cd continue
git checkout main

:: télécharger les 4 patchs depuis ce repo (navigateur OK) :
::   raw.githubusercontent.com/samskinov/csharp-examples/main/<fichier>.patch
git am continue-improvements.patch
git am continue-cli-modes.patch
git am continue-fallbacks.patch
git am continue-cli-tools.patch
:: (secours si git am échoue : git apply -3 <fichier>)
```

⚠️ **Base exacte obligatoire** : les patchs sont générés sur l'état final de
`main` (commit `5522c6f` — « docs: remove Sign in link… »). **Ne pas utiliser le
tag `v2.1.0-vscode`** : il pointe quelques commits plus tôt et certains hunks ne
s'appliquent pas. Test décisif après checkout :

```cmd
git log -1
:: doit afficher : 5522c6f docs: remove Sign in link (login flow retired) (#13005)
```

Si un `git am` a été interrompu auparavant, purger l'état résiduel :
```cmd
git am --abort 2>nul
rmdir /s /q .git\rebase-apply 2>nul
git reset --hard refs/remotes/origin/main
```

## 2. Installs + builds (registry npm uniquement)

⚠️ `scripts\build-packages.js` ne construit **que** `packages\*` — il ne touche
ni `core`, ni `gui`, ni les extensions. Après un `git clean -xfd` (qui supprime
tous les `node_modules`), `core` doit donc être **réinstallé et recompilé
explicitement**. Sans ça, le `tsc` de `gui` échoue avec des dizaines de
« Cannot find module ... » qui pointent vers `..\core\...` : gui résout les
types de `core` via `core\index.d.ts`, qui a besoin des dépendances de core.

```cmd
:: 1) packages internes (config-types, terminal-security, fetch, ...)
node .\scripts\build-packages.js

:: 2) core — OBLIGATOIRE avant gui
::    --include=dev force les devDependencies (shiki, typescript, ...)
::    --ignore-scripts évite les téléchargements hors registry npm
::    (sharp/libvips et Chrome de puppeteer, inutiles au build) :
cd core && npm ci --include=dev --ignore-scripts --no-audit --no-fund && npm run build && cd ..

:: 3) gui et extensions : installation des dépendances
cd gui && npm ci --no-audit --no-fund && cd ..
cd extensions\vscode && npm ci --no-audit --no-fund && cd ..\..
cd extensions\cli && npm ci --no-audit --no-fund && cd ..\..
```

`npm ci` installe **exactement** les `package-lock.json` du patch et repart d'un
`node_modules` propre. Pour ajouter ou modifier une dépendance, faites un
`npm install` une fois (pour mettre à jour le lock), puis revenez à `npm ci`.

⚠️ En CMD, `&` **continue après un échec** ; `&&` s'arrête au premier échec.
Chaque ligne ci-dessus est une étape : si elle renvoie un code ≠ 0
(`echo %ERRORLEVEL%`), corrigez-la avant de passer à la suivante.

Aucun module natif à compiler pour le build : sqlite3 est remplacé par
`node-sqlite3-wasm` (WASM depuis le registry), la recherche par une
implémentation intégrée. Le provider `@database` (déprécié) est désactivé : il
tirait `dbinfoz`, donc `sqlite3` et son binding natif.

### Réinstallation propre (node_modules incomplet)

Symptômes : `Cannot find module 'shiki'`, `'safer-buffer'`, `'symbol-tree'`, ...
alors que le dossier semble présent dans `node_modules`. `npm install` est
incrémental et peut laisser un arbre partiel après une installation
interrompue ; `npm ci` repart de zéro et échoue franchement si un paquet ne peut
pas être téléchargé.

```cmd
:: (si les packages internes ne sont pas construits :)
node .\scripts\build-packages.js

cd core
rmdir /s /q node_modules
npm ci --include=dev --ignore-scripts --no-audit --no-fund
echo CORE_INSTALL=%ERRORLEVEL%
npm run build

cd ..\gui
rmdir /s /q node_modules
npm ci --no-audit --no-fund
npm run build

cd ..\extensions\vscode
rmdir /s /q node_modules
npm ci --no-audit --no-fund
echo VSCODE_INSTALL=%ERRORLEVEL%
npm run prepackage -- --target win32-x64
npm run package -- --target win32-x64
```

Vérification rapide côté core :

```cmd
cd core
node -p "require.resolve('shiki/package.json')"
node -p "require.resolve('safer-buffer/package.json')"
node -p "require.resolve('symbol-tree/package.json')"
```

Si `npm ci` échoue avec `EINTEGRITY` : `npm cache verify`, puis si besoin
`npm cache clean --force`, et relancer. Si l'échec est un 407/403 sur un
tarball, c'est ce paquet qu'il faut autoriser sur le proxy.

## 3. Build de l'extension VS Code (VSIX)

```cmd
cd gui && npm run build && cd ..
::   tsc + vite -> gui\dist

cd extensions\vscode
:: dépendances déjà installées par npm ci : évite un npm install réseau
set SKIP_INSTALLS=true
npm run prepackage -- --target win32-x64
npm run package -- --target win32-x64
cd ..\..
::   -> extensions\vscode\build\continue-<version>.vsix

:: installation :
code --install-extension extensions\vscode\build\continue-<version>.vsix
```

## 4. Build du CLI (`cn`)

```cmd
cd extensions\cli
npm run build                      :: -> dist\cn.js (bundle autonome)

node dist\cn.js --help             :: vérif : flags --mode / --tool-mode
node dist\cn.js --mode chat
node dist\cn.js --mode plan --tool-mode text
npm link                           :: (optionnel) commande `cn` globale
```

Modes de permission (qui décident quels tools sont proposés/exécutés) :

| Mode | Comportement |
|---|---|
| `normal` (défaut) | agent, demande confirmation avant les tools/actions |
| `plan` | lecture seule (`--readonly`), aucune modification |
| `auto` | tous les tools autorisés sans confirmation (`--auto`) |
| `chat` | conversation simple, **aucun tool natif déclaré** |

Bascule en session avec **Shift+Tab**, ou au lancement avec `--mode <mode>`.
`--tool-mode native|text` est indépendant : `native` utilise le paramètre
`tools` d'OpenAI, `text` décrit les tools dans le system message et lit les
appels dans la réponse (utile quand le function calling du serveur est
instable). En session, `/tool-mode` (visible dans la liste `/`) bascule entre
les deux ; en mode `chat`, aucun tool n'est envoyé, même avec `native`.

En headless (`-p`), il n'y a pas de confirmation interactive : les tools en
politique `ask` ne sont pas proposés. Utiliser `--allow Read` (etc.) pour en
autoriser un précis, ou `--auto`.

## 5. Notes

- **« Tool ... not found » avec un nom terminé par `</arg_value>`** : certains
  serveurs OpenAI-compatibles (GLM via litellm/Ollama) laissent fuiter la
  syntaxe XML native du modèle dans `tool_calls`. Le patch #4 nettoie le nom
  (`Read</arg_value>` → `Read`), enlève le tag après le JSON, parse les paires
  `<arg_key>/<arg_value>` et corrige les alias d'arguments courants
  (`file_path` → `filepath`). Côté VS Code, le même nettoyage est appliqué au
  streaming, à l'historique et aux tools client.
- **`400 litellm.BadRequestError: ... system message must be at the beginning`**
  : le patch #4 supprime les system messages ajoutés en cours d'historique
  (erreurs, annulation de tool, compaction) et n'envoie qu'un seul system
  message, en première position. Vérifié contre un proxy strict et Ollama.
- **« Could not locate the bindings file » / `node_sqlite3.node` à l'ouverture
  de VS Code** : l'ancien patch #3 laissait `dbinfoz` (donc `sqlite3`) dans les
  dépendances de l'extension ; esbuild l'embarquait et l'activation échouait.
  Le patch #3 corrigé le retire. Si l'ancien patch #3 est déjà appliqué et
  qu'aucun patch #4 n'est par-dessus :
  ```cmd
  git reset --hard HEAD^
  git am continue-fallbacks.patch

  cd core && npm ci --include=dev --ignore-scripts --no-audit --no-fund && npm run build && cd ..
  cd extensions\vscode && npm ci --no-audit --no-fund
  npm run prepackage -- --target win32-x64 && npm run package -- --target win32-x64
  ```
  Si le patch #4 a déjà été appliqué par-dessus l'ancien #3, remplacez
  `HEAD^` par `HEAD~2`, puis réappliquez `continue-fallbacks.patch` **et**
  `continue-cli-tools.patch` dans cet ordre.

- **Node 24** validé de bout en bout (v24.22.3) : `engines` >= 20.20.1 ✓.
- Le **tag nommé `main`** du repo rend `main..branche` ambigu : utiliser
  `refs/heads/main..` dans les comparaisons git.
- **« Cannot find module 'lru-cache' / 'web-tree-sitter' / ... »** dans le build
  de `gui`, avec des chemins `..\core\...` : les dépendances de `core` ne sont
  pas installées. Refaire l'étape 2 (`cd core && npm install ...`). C'est le
  symptôme typique d'un `git clean -xfd` suivi de `build-packages.js` seul.
- **« Cannot find module 'shiki' »** dans `core` : `shiki` est une
  devDependency de core, importée par `core/codeRenderer/CodeRenderer.ts` et
  `core/nextEdit/diff/diff.ts` — tsc les compile même s'ils ne sont pas
  utilisés au runtime. Si elle manque, l'install de core n'a pas abouti ou les
  devDependencies ont été omises (`NODE_ENV=production`, `--omit=dev`).
  Relancer `npm ci --include=dev --ignore-scripts` puis vérifier
  `dir node_modules\shiki`. Si elle reste absente, forcer :
  `npm install shiki@3.23.0 --save-dev --ignore-scripts --no-audit --no-fund`
  puis vérifier `node -p "require('./node_modules/shiki/package.json').version"`.
  Si cette install échoue, la vraie cause est dans son log (chemin de clone
  trop long sous Windows, tarball bloqué par le proxy, disque plein) — shiki
  n'est que le premier module manquant.
- Le `--ignore-scripts` de l'étape 2 évite deux téléchargements hors registry
  npm faits par des dépendances de `core` : le libvips de `sharp` (GitHub) et
  le Chrome de `puppeteer` (Google). Ils ne servent pas au build : le code
  Transformers.js vendoré n'importe pas `sharp`, et l'indexation de docs avec
  puppeteer est optionnelle.
- L'esbuild embarqué du VSIX vient du **bucket S3 AWS**
  (`continue-server-binaries.s3.us-west-1.amazonaws.com`) — souvent autorisé
  par les proxies corporate. Si bloqué : le VSIX fonctionne, seule la
  compilation de `config.ts` custom à l'exécution est absente.
- Télémétrie : **strictement opt-in** (`CONTINUE_CLI_ENABLE_TELEMETRY=1` +
  endpoint OTel) — rien ne part par défaut.
