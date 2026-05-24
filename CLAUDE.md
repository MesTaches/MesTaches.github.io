# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Nature du projet

Application web **monofichier HTML** (`index.html`) : application React 18 + Babel-standalone compilée **en-browser**. **Aucun build, aucun bundler, aucun test framework.** Le JSX vit dans un `<script id="jsx-source" type="text/plain">` et est transformé par Babel au chargement de la page.

Repo déployé via **GitHub Pages** sur `MesTaches.github.io` — push sur `main` = mise en production immédiate.

Le numéro de version est dans `<meta name="app-version">` (en haut) et dans le footer de la sidebar (`v.0.NNN`). La version doit être incrémentée à chaque commit qui modifie l'app.

## Workflow

Pas de commande de build, pas de tests. Workflow standard :

```powershell
# Tester localement
start index.html     # ouvre dans le navigateur par défaut

# Pousser une modif
git add index.html
git commit -m "fix: <description>"
git push             # déploiement GitHub Pages automatique
```

**Détection d'erreur** : ouvrir `index.html` dans Chrome, F12 → Console.
- **Page blanche complète** = erreur de syntaxe JSX (Babel-standalone n'a pas de récupération d'erreur) **OU** exception runtime dans un composant sans Error Boundary (React démonte tout l'arbre).
- L'erreur exacte est toujours dans la console Chrome.

**Reset des données locales pendant un debug** :
```js
indexedDB.deleteDatabase("app_main_db")
indexedDB.deleteDatabase("creditbail_db")
```

## Architecture en couches

### Authentification & sync (avant React)
`<script>` en tête de `<body>` (avant le bundle React) :
- Firebase Auth gère la connexion email/mot de passe (projet `mes-taches-632e4`).
- Une fois loggé, `loadAndMount(uid)` recharge les `SYNC_KEYS` depuis Firestore vers localStorage, puis monte React.
- `Storage.prototype.setItem` est **monkey-patché** : toute écriture dans une clé `SYNC_KEYS` est répliquée vers Firestore (`users/{uid}/ls/{docKey}`) avec un debounce de 1 s.
- Un `setInterval(10s)` détecte les changements distants et dispatch l'événement `remoteDataUpdated` à React (qui relit localStorage).

### Stockage local
**Deux IndexedDB distincts** :
- `app_main_db` (helpers `appDbGet(store)` / `appDbSet(store, data)`) — stores : `tasks`, `tasks2`, `dev`, `contentieux`, `cotations`, `payments`, `reports`, `echeancier`, `rcf`, `suivi_clients`, `meetings`, `responsables`, `settings`, `user_profiles`, et leurs corbeilles (`trash_*`).
- `creditbail_db` — IndexedDB **séparé** pour les pièces jointes du crédit-bail (volumes > 900 Ko qui ne tiendraient pas dans localStorage). Helpers locaux : `openCBDB`, `idbGetAll`, `idbPut`, `idbDelete`.

**LocalStorage** sert de pont legacy : `LS_TO_STORE` mappe les anciennes clés (`taskmanager_v4`, `taskmanager_meetings_v1`…) vers les nouveaux stores IDB, avec migration automatique au premier lancement (`migrateAllToIDB`).

Hook React `useAppStore(store, default)` = `useState` + chargement IDB + debounced write 300 ms.

### Convention JSX & utilitaires globaux

**Polices** (string CSS) :
- `_FL = "'Lora',serif"` — titres, texte courant
- `_DM = "'DM Mono',monospace"` — petites capitales, nombres tabulaires

**Styles partagés** dans `const S` :
- `S.input` — input standard
- `S.btn(bg, color = '#fff')` — bouton plein
- `S.ghost` — bouton ghost en pointillés

**Modales** : utiliser `ModalSection({title, children})`, `_MODAL_LBL(text)` (renvoie un `<div>` label stylé), `_MODAL_INP` (objet de style).

**Composants utilitaires** :
- `RichTextArea` — éditeur contentEditable avec toolbar (gras, couleur, listes, IA Groq optionnelle).
- `useIsMobile()` — `window.innerWidth < 640`.
- `InlineSelect`, `SubStatusSelect` — dropdowns custom stylés.
- `AttachmentZone` — drag & drop fichiers (limite 5 Mo, base64).

**Helpers ID & dates** :
- `uid()` — random 9 chars.
- `today()` — `YYYY-MM-DD` du jour.
- `fmtDate(d)` — format `JJ mois AAAA` FR.
- `parseNum(v)` — parse nombre avec virgule/espaces/apostrophes.

### Structure d'un module-vue typique

Chaque grand onglet (Réunions, Cotation, RCF, Crédit-bail, Échéancier, Contentieux, Dev, Suivi clients…) est un composant React avec :

1. **Constantes locales** : statuts, priorités, couleurs (préfixe par module : `MTG_*`, `CB_*`, `RCF_*`…).
2. **Factory functions** : `_mkMtg`, `_mkSess`, `_mkAct` pour créer des objets vides.
3. **State local** : `useState` + `appDbGet`/`appDbSet` direct (pas `useAppStore` partout — pattern historique).
4. **Modal d'édition** (ex : `MeetingActionPanel`, `CBDossierModal`) — composant séparé qui reçoit l'objet, l'édite localement, appelle `onSave`/`onClose`.
5. **Sidebar/Kanban/Tableau** selon la nature du module.

### Routing
Pas de router : `const [view, setView] = useState("dashboard")` dans `App`. La sidebar gauche déclenche `setView(id)`. Le rendu est un gros chain `{view === "xxx" && <XxxView />}` dans `App.render`.

**Deux conteneurs** dans `App` (le ternaire est ~ligne 14648) :
- Bloc **full-width** (Tâches, Catégories, Calendrier, Réunions, Crédit-bail, Échéancier, etc.)
- Bloc **maxWidth contenu** (Dashboard, Kanban, Notifications, Paramètres…)

**Si un nouvel onglet ne s'affiche pas** (écran blanc côté droit) : vérifier qu'il est rendu **dans le bon bloc** ET que sa valeur `view` apparaît dans la condition du ternaire. Ce bug est arrivé sur Réunions (cf. git log).

### Outlook Add-in

Le repo héberge un add-in Office :
- `manifest.xml` — le manifeste sideloadé dans Outlook
- `outlook-addin.html` — la page qui s'exécute dans le task pane Outlook
- `icon-16.png`, `icon-32.png`, `icon-80.png` — icônes du bouton ribbon

Flux :
1. Utilisateur clique le bouton **→ MesTaches** dans Outlook (mode lecture d'un email)
2. Office.js lit `subject`, `body` (texte plat), `from`, `dateTimeCreated` de l'email courant
3. La page ouvre `https://mestaches.github.io/?newTask=1&title=…&from=…&date=…&notes=…`
4. Dans `App` (`index.html`), un `useEffect([appReady])` détecte le paramètre `newTask=1`, appelle `openNew({title, responsable, notes})` et nettoie l'URL via `history.replaceState`

Le `notes` est tronqué à 1500 caractères côté add-in pour rester sous la limite de longueur URL.

**Modifier l'URL cible** : si tu changes de domaine, met à jour les `https://mestaches.github.io/` dans `manifest.xml` ET dans `outlook-addin.html` (constante `APP_URL`).

**Sideload du manifest** : Outlook → ⋯ → Get Add-ins → My add-ins → Custom Addins → "Add from URL" → `https://mestaches.github.io/manifest.xml`.

### PWA / Service Worker

`sw.js` est minimal — désinstallation auto au chargement (`<head>` script). Pas de cache offline. Si on en ajoute un, attention : il devra invalider à chaque nouvelle version.

## Conventions de modification importantes

- **Setter d'objet** : toujours `setD(p => ({...p, field: val}))`, jamais `setD({...d, field: val})` (React peut batcher et perdre des updates).
- **JSX balisé** : toute balise doit être strictement équilibrée — Babel-standalone n'a pas de récupération d'erreur. Une `</div>` manquante = page blanche.
- **Pas de Fragment React** dans le code existant — utiliser un `<div>` wrapper.
- **Données legacy** : toujours normaliser au chargement (`arr.map(m => ({...defaults, ...m}))`). Des objets sans certaines propriétés circulent en IDB de versions antérieures.
- **Incrémenter la version** dans `<meta name="app-version">` ET le footer sidebar (`v.0.NNN`) à chaque commit qui modifie l'app.

## Sync Firebase (clés à connaître)

`SYNC_KEYS` dans le script Firebase liste les clés localStorage répliquées vers Firestore :
```
taskmanager_v4, taskmanager_settings_v1, taskmanager_meetings_v1,
devtracker_tasks_v1, devtracker_settings_v1,
contentieux_dossiers_v1, contentieux_settings_v1
```

**Tout module qui doit être synchronisé entre appareils** doit écrire dans une de ces clés legacy (en plus de IDB). Sinon les données restent locales à un appareil.

## Pièges connus

- **Apostrophes typographiques** (`'`) dans les attributs JSX cassent parfois Babel-standalone. Utiliser `'` ASCII.
- **`Object.assign({}, p, {[k]: v})`** avec computed key fonctionne. Le spread `{...p, [k]: v}` aussi. Mais éviter de mélanger les deux dans la même fonction.
- **`window.parseCSVEcheancier` / `window.parseXLSXEcheancier`** sont exposés sur `window` car appelés depuis des callbacks FileReader hors React.
- **Crédit-bail** : si une PJ dépasse ~900 Ko, basculer sur IDB `creditbail_db` (champ `dataRef` au lieu de `data` inline).
- **`MEETING_STORAGE` = `"taskmanager_meetings_v1"`** (clé localStorage/sync) est **différent de** **`MTG_KEY` = `"meetings"`** (nom du store IDB). Les deux pointent vers la même donnée mais par deux chemins.
- **Fichier de 1 Mo** : les outils de recherche (Grep) peuvent retourner beaucoup de résultats — préférer des patterns précis avec contexte.
