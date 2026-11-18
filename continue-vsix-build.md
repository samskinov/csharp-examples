# Installer / compiler Continue (VS Code + CLI) avec les patchs

Ce repo fournit **3 patchs** à appliquer sur `continuedev/continue` (v2.1.0 ou
main) qui rendent le build et l'exécution complets **sans aucun téléchargement
en dehors du registry npm** (rien depuis GitHub, nodejs.org ou S3) — pensés
pour les postes derrière un proxy d'entreprise.

## Les 3 patchs (dans cet ordre)

| Ordre | Fichier | Contenu |
|---|---|---|
| 1 | `continue-improvements.patch` | 14 commits : résilience stream/thinking/429, freezes (tokenisation workers, sessions async), tools, watchdog |
| 2 | `continue-cli-modes.patch` | 4 commits : CLI chat/plan/agent + tools text, streaming live, perf GUI |
| 3 | `continue-fallbacks.patch` | 1 commit : SQLite WASM (plus de binding natif), recherche intégrée (plus de ripgrep), prepackage adapté |

⚠️ Vérification faite : empilement `git am` 14+4+1 = **19 commits** sur `main`
propre. Version de référence : `v2.1.0-vscode` (tag) ou `main` (état final).

## 1. Clone + patchs

```cmd
git clone https://github.com/continuedev/continue continue
cd continue
git checkout v2.1.0-vscode

:: télécharger les 3 patchs depuis ce repo (navigateur OK) :
::   raw.githubusercontent.com/samskinov/csharp-examples/main/<fichier>.patch
git am continue-improvements.patch
git am continue-cli-modes.patch
git am continue-fallbacks.patch
:: (secours si git am échoue : git apply -3 <fichier>)
```

## 2. Installs (registry npm uniquement)

```cmd
:: packages internes (ordre géré par le script officiel) :
node .\scripts\build-packages.js

:: par dossier (si besoin de repasser) :
cd packages\config-types & npm install & npm run build & cd ..\..
cd packages\terminal-security & npm install & npm run build & cd ..\..
cd packages\llm-info & npm install & npm run build & cd ..\..
cd packages\config-yaml & npm install & npm run build & cd ..\..
cd packages\fetch & npm install & npm run build & cd ..\..
cd packages\openai-adapters & npm install & npm run build & cd ..\..
cd core & npm install & npm run build & cd ..\..

cd gui & npm install & cd ..
cd extensions\vscode & npm install & cd ..\..
cd extensions\cli & npm install & cd ..\..
```

Aucun module natif à compiler : sqlite3 est remplacé par `node-sqlite3-wasm`
(WASM depuis le registry), la recherche par une implémentation intégrée,
l'esbuild embarqué vient du bucket S3 (si bloqué : voir note § 5).

## 3. Build de l'extension VS Code (VSIX)

```cmd
cd gui
npm run build                      :: tsc + vite -> gui\dist
cd ..\extensions\vscode
npm run prepackage -- --target win32-x64
npm run package -- --target win32-x64
::   -> build\continue-<version>.vsix

:: installation :
code --install-extension build\continue-<version>.vsix
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

## 5. Notes

- **Node 24** validé de bout en bout (v24.22.3) : `engines` >= 20.20.1 ✓.
- Le **tag nommé `main`** du repo rend `main..branche` ambigu : utiliser
  `refs/heads/main..` dans les comparaisons git.
- L'esbuild embarqué du VSIX vient du **bucket S3 AWS**
  (`continue-server-binaries.s3.us-west-1.amazonaws.com`) — souvent autorisé
  par les proxies corporate. Si bloqué : le VSIX fonctionne, seule la
  compilation de `config.ts` custom à l'exécution est absente.
- En cas d'erreur résiduelle (mirror d'entreprise, DLP), les recettes
  détaillées par téléchargeur sont dans l'historique git de ce fichier
  (versions antérieures) — mais avec les 3 patchs elles ne devraient plus
  être nécessaires.
- Télémétrie : **strictement opt-in** (`CONTINUE_CLI_ENABLE_TELEMETRY=1` +
  endpoint OTel) — rien ne part par défaut.
