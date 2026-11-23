# Installer / compiler Continue (VS Code + CLI) avec les patchs

Ce repo fournit **7 patchs** à appliquer sur `continuedev/continue` (v2.1.0 ou
main) qui suppriment les dépendances aux binaires natifs (SQLite natif,
ripgrep), rendent le build reproductible depuis le registry npm seul et
corrigent l'activation Windows — pensés pour les postes derrière un proxy
d'entreprise.

## Les 7 patchs (dans cet ordre)

| Ordre | Fichier | Contenu |
|---|---|---|
| 1 | `01-continue-improvements.patch` | 14 commits : résilience stream/thinking/429, freezes, tools, watchdog |
| 2 | `02-continue-cli-modes.patch` | 4 commits : CLI chat/plan/agent + tools text, streaming live, perf GUI |
| 3 | `03-continue-fallbacks.patch` | 1 commit : SQLite WASM, plus de `dbinfoz`/`@database`, recherche intégrée, prepackage |
| 4 | `04-continue-cli-tools.patch` | 6 commits : chat sans tools natifs, system message unique, `/tool-mode`, tool calls XML, fallback texte, VSIX ciblé |
| 5 | `05-continue-vscode-activation.patch` | 1 commit : logs d'activation, `system-ca` dynamique, binding `win-ca` |
| 6 | `06-continue-agent-performance.patch` | 1 commit : outils agent, boucles et streaming texte |
| 7 | `07-continue-text-tools.patch` | 1 commit : mode texte fiable OpenAI-compatible, aucun `tools`/`tool_calls` natif, schémas/exemples, réparation/validation des arguments, alias CLI↔GUI, rappel syntaxe+exemple dans les erreurs, `/cd` persistant et restauré au resume/fork, ChangeDirectory, MCP prêt avant la première requête, virtualisation maison de l’historique GUI sans nouvelle dépendance |

⚠️ Empilement vérifié : `git am` 14+4+1+6+1+1+1 = **28 commits** sur
`refs/heads/main` propre.

## 1. Clone + patchs

```cmd
git clone https://github.com/continuedev/continue continue
cd continue
git checkout main

:: télécharger les 7 patchs depuis ce repo (navigateur OK) :
::   raw.githubusercontent.com/samskinov/csharp-examples/main/<fichier>.patch
git am 01-continue-improvements.patch
git am 02-continue-cli-modes.patch
git am 03-continue-fallbacks.patch
git am 04-continue-cli-tools.patch
git am 05-continue-vscode-activation.patch
git am 06-continue-agent-performance.patch
git am 07-continue-text-tools.patch
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

### Si `npm ci` a déjà été exécuté correctement

Après l'application du patch 06 ou de toute autre modification source, il n'est
pas nécessaire de refaire `npm ci` si les fichiers `package.json`,
`package-lock.json` et l'arbre `node_modules` n'ont pas changé.

Les dépendances déjà installées restent utilisables. Reconstruire seulement les
couches dont le code source a été modifié:

```cmd
:: Depuis la racine continue\
cd core
npm run build
cd ..

cd gui
npm run build
cd ..

cd extensions\cli
npm run build
cd ..\..

cd extensions\vscode
set SKIP_INSTALLS=true
npm run prepackage -- --target win32-x64
npm run package -- --target win32-x64
```

Avec le VS Code GUI patché, `gui` doit être construit avant
`extensions\vscode`, sinon le VSIX peut embarquer une ancienne version du GUI.

Relancer `npm ci` uniquement dans ces cas:

```text
- package.json ou package-lock.json ont été modifiés;
- node_modules est incomplet ou une erreur Cannot find module apparaît;
- l'installation précédente a été interrompue;
- core/gui/cli/vscode doivent repartir d'une arborescence propre.
```

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

Le VSIX embarque `config-yaml-schema.json` et les schémas : **aucune
compilation supplémentaire de la config n'est nécessaire**. Si VS Code affiche
`Unable to write to User Settings because yaml.schemas is not a registered
configuration`, c'est seulement l'extension YAML (redhat.vscode-yaml) qui n'est
pas installée ; Continue continue de s'activer normalement.

### Voir l'erreur d'activation (Windows)

Le bouton « View Logs » de l'ancien VSIX ne faisait rien : il appelait
`continue.viewLogs`, une commande enregistrée seulement après une activation
réussie. Avec le patch #5 :

1. Le message d'erreur affiche maintenant le texte réel.
2. Un canal **Continue Activation** contient la stack complète.
3. Le fichier `%USERPROFILE%\.continue\logs\activation-error.log` est écrit.

Pour voir l'erreur dans VS Code, quelle que soit la version :

```text
Ctrl+Shift+P → Developer: Show Logs... → Extension Host
```

ou :

```text
Help → Toggle Developer Tools → Console (chercher "Continue")
```

ou : `Ctrl+Shift+P` → `Developer: Open Extension Logs Folder`, puis ouvrir
`exthost.log` / `window1\exthost\exthost.log` et chercher `Continue`.
Le log « Continue » de la barre latérale n'est pas le log d'activation.

Un VSIX Windows x64 déjà compilé avec ces correctifs est disponible ici :
`github.com/samskinov/csharp-examples/releases/tag/continue-vscode-1.3.40-win32`.

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

### OpenAI-compatible : mode texte obligatoire

Si le proxy refuse toute requête contenant `tools`, utiliser le protocole
texte (c'est le défaut du CLI) :

```cmd
node dist\cn.js --tool-mode text ...
```

Le CLI n'envoie alors **ni `tools`, ni `tool_calls` dans l'historique, ni
messages `role: tool`** : les appels sont dans des blocs Markdown et les
résultats sont renvoyés dans des blocs `tool_result` en rôle `user`.

Côté extension VS Code, le mode texte est sélectionné automatiquement pour
`ollama` et pour les endpoints OpenAI personnalisés (`apiBase` ou
`onPremProxyUrl`). Pour forcer globalement :

```yaml
experimental:
  onlyUseSystemMessageTools: true
```

Le modèle reçoit alors les définitions et exemples de tools dans le system
message, avec les paramètres requis et un exemple concret par outil. Si un
appel est invalide, Continue renvoie le schéma et un exemple corrigé au lieu
d'exécuter l'appel ; le modèle doit corriger ses arguments au lieu de répéter
la même erreur.

`ApplyPatch` est l'outil d'édition privilégié pour les fichiers existants ;
`AskQuestion` est l'outil à utiliser pour toute clarification ; `ChangeDirectory`
change réellement le répertoire de travail persistant (un `cd` dans Bash ne
dure qu'une commande).

## 5. Notes

- **MCP jaune / tools MCP invisibles au premier message** : le handshake MCP
  est asynchrone. Continue attend maintenant jusqu'à 8 s avant la première
  requête et recharge la config quand les tools arrivent. Si le serveur reste
  jaune, le problème est côté serveur MCP (commande, proxy, npx) : le stderr
  est dans l'erreur du panneau MCP et dans le log Extension Host.
- **Freeze VS Code pendant « generating »** : la webview ne rend plus tout
  l'historique à chaque token. Les messages hors écran utilisent
  `content-visibility`, seuls les 80 derniers sont montés par défaut (bouton
  « Show earlier messages »), et un stream silencieux est maintenant retenté
  au bout de 60 s avec un message visible au lieu de laisser « generating »
  indéfiniment.
- **`/cd` dans le CLI** : le répertoire change réellement (`process.chdir`),
  et la session, l'index de fichiers et les hooks sont mis à jour. Le tool
  `ChangeDirectory` fait la même chose pour le modèle ; un `cd` dans Bash ne
  dure qu'une commande.

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
  git am 03-continue-fallbacks.patch

  cd core && npm ci --include=dev --ignore-scripts --no-audit --no-fund && npm run build && cd ..
  cd extensions\vscode && npm ci --no-audit --no-fund
  npm run prepackage -- --target win32-x64 && npm run package -- --target win32-x64
  ```
  Si le patch #4 et/ou #5 ont déjà été appliqués par-dessus l'ancien #3,
  remplacez `HEAD^` par `HEAD~2` (patch #4) ou `HEAD~3` (patch #4 + #5), puis
  réappliquez `03-continue-fallbacks.patch`, `04-continue-cli-tools.patch` et
  `05-continue-vscode-activation.patch` dans cet ordre.

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

## 6. Node 24

Node 24 est le runtime de validation recommandé:

```cmd
node --version
:: v24.x
npm --version
```

Toutes les étapes de build doivent utiliser le même Node 24 pour core, gui,
CLI et VSIX. Aucun binding natif SQLite/ripgrep n'est requis:

- SQLite utilise `node-sqlite3-wasm`;
- les recherches intégrées n'utilisent pas un binaire ripgrep obligatoire;
- LanceDB VSIX Windows est limité à win32-x64;
- les scripts `--ignore-scripts` évitent les téléchargements Sharp/Puppeteer
  sans bloquer le build.

