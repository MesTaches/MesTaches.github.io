# Architecture de `index.html`

> Carte de navigation dans le mono-fichier (~1 Mo). Les numéros de ligne **vont bouger**, prends-les comme indication d'ordre relatif. Cherche par nom plutôt que par ligne (`Grep` ou Ctrl+F).

## Organisation top-down

```
index.html
├─ <head>                           ── meta version, favicons
├─ <body>
│   ├─ <script>                     ── Firebase Auth + sync localStorage ↔ Firestore
│   │   • SYNC_KEYS                 ── liste des clés legacy répliquées
│   │   • loadAndMount(uid)         ── recharge depuis Firestore, monte React
│   │   • monkey-patch setItem      ── intercept écriture localStorage
│   │
│   ├─ <script id="jsx-source"      ── tout le code React (compilé par Babel-standalone)
│   │   type="text/plain">
│   │
│   │   ┌── Helpers globaux ──────────────────────────────
│   │   • DEFAULT_SETTINGS, loadSettings()
│   │   • appDbGet / appDbSet      ── IDB app_main_db
│   │   • openCBDB / idbPut        ── IDB creditbail_db (PJ > 900 Ko)
│   │   • useAppStore(store, def)  ── hook React → IDB
│   │   • _FL, _DM, S.btn(), S.input
│   │   • uid(), today(), fmtDate, parseNum, isOverdue, isToday
│   │   • _MODAL_LBL, _MODAL_INP, ModalSection
│   │
│   │   ┌── Composants UI réutilisables ──────────────────
│   │   • InlineSelect, SubStatusSelect, ColorMenu, HighlightMenu, BulletMenu
│   │   • RichTextArea               ── éditeur contentEditable + IA Groq
│   │     - RichTextAiBtn            ── menu IA (corriger / reformuler / résumer)
│   │     - Resize images intégré (clic sur image → poignée bas-droite)
│   │   • AttachmentZone             ── drag & drop, base64, limite 5 Mo
│   │   • DailySummary               ── modale au 1er lancement du jour
│   │
│   │   ┌── Modules-vues (gros composants) ────────────────
│   │   • CotationView               ── grand tableau cotations clients
│   │   • Dashboard                  ── tableau de bord + panneau Aujourd'hui
│   │   • NotificationsView          ── alertes RCF / Crédit-bail / Réunions
│   │   • CalendarView               ── calendrier mensuel
│   │   • MeetingsView               ── liste réunions + RichTextArea
│   │     - MeetingPanel             ── header + zone notes
│   │     - MeetingSettingsPanel     ── modale paramètres réunion
│   │     - MeetingNewModal          ── création
│   │   • DevView                    ── suivi évolutions app
│   │   • TasksView                  ── vue principale (tableau + cartes)
│   │     - TaskModal                ── modale latérale d'édition tâche
│   │     - SubtaskDetailModal       ── détail d'une sous-tâche
│   │   • CategoriesView             ── vue catégories en colonnes
│   │   • KanbanView                 ── kanban à colonnes statut
│   │   • RcfView                    ── lignes de crédit + tirages
│   │   • CreditBailView             ── dossiers + financeurs + PJ
│   │   • EcheancierView             ── prêts + lignes d'échéances (CSV/XLSX)
│   │   • ContentieuxView            ── dossiers contentieux
│   │   • PaymentsView               ── suivi paiements
│   │   • SuiviClientsView           ── suivi commercial clients
│   │   • SettingsModal              ── paramètres globaux (catégories, Groq)
│   │
│   │   ┌── App() ── ROOT ────────────────────────────────
│   │   • useState pour tasks, settings, view, modales
│   │   • Routing : {view === "xxx" && <XxxView/>}
│   │   • Capture Outlook newTask=1 via URLSearchParams
│   │   • Sidebar (rendu inline dans App)
│   │   • Footer : version v.0.NNN
│   │
│   └─ <script id="..." />ReactDOM.render(<App/>)
└─
```

## Stockage : quelle clé pour quoi

### IndexedDB `app_main_db` (helper `appDbGet/Set`)
| Store           | Contenu                                | Sync Firestore ?       |
|-----------------|----------------------------------------|------------------------|
| `tasks`         | Tâches principales                     | Via `taskmanager_v4`   |
| `tasks2`        | Sous-tâches éclatées (legacy)          | Non                    |
| `dev`           | Évolutions de l'app (onglet Dev)       | Via `devtracker_…`     |
| `contentieux`   | Dossiers contentieux                   | Via `contentieux_…`    |
| `cotations`     | Cotations clients                      | Non (local)            |
| `payments`      | Suivi paiements                        | Non (local)            |
| `reports`       | Rapports                               | Non                    |
| `echeancier`    | Prêts                                  | Non                    |
| `rcf`           | RCF (contrats + tirages)               | Via `taskmanager_rcf_v1` (lecture seule) |
| `suivi_clients` | Suivi commercial                       | Non                    |
| `meetings`      | Réunions (clé `MTG_KEY`)               | Via `taskmanager_meetings_v1` |
| `responsables`  | Liste des responsables                 | Non                    |
| `settings`      | Settings de l'app                      | Via `taskmanager_settings_v1` |
| `user_profiles` | Profils utilisateurs                   | Non                    |
| `trash_*`       | Corbeille par module (30 jours)        | Non                    |

### IndexedDB `creditbail_db`
- Stocke les **pièces jointes** du crédit-bail (`> 900 Ko` qui ne tiendraient pas dans localStorage)
- Helpers locaux : `openCBDB`, `idbGetAll`, `idbPut`, `idbDelete`
- Référencement : champ `dataRef` (id) dans la PJ au lieu de `data` (base64 inline)

### LocalStorage
- **Pont legacy** : `LS_TO_STORE` mappe les anciennes clés vers les nouveaux stores IDB
- **Corbeilles** :
  - `deletedTasks` — tâches supprimées (30 j)
  - `deletedMeetings` — réunions supprimées (30 j)
  - `cotation_trash_v1` — cotations supprimées
- **Settings divers** : largeurs de colonnes, dernière date de résumé du jour, etc.

## Conventions à respecter

Voir `CLAUDE.md` pour la liste complète. Les plus importantes :

1. **Pas de Fragment React** — utiliser `<div>` wrapper.
2. **Apostrophes ASCII** uniquement dans les attributs JSX (`'` pas `'`).
3. **Setter d'objet** : `setD(p => ({...p, field: val}))` toujours.
4. **Données legacy** : normaliser au chargement (`arr.map(x => ({...defaults, ...x}))`).
5. **Incrémenter la version** à chaque commit qui modifie l'app.
6. **Tester localement** avant push (`start index.html`, F12 console).
7. **Babel-standalone n'a pas de récupération** — une syntaxe cassée = écran blanc complet.

## Si on voulait découper le fichier (un jour)

Le mono-fichier devient inconfortable au-delà de ~1.5 Mo. Pistes envisageables, par ordre de complexité croissante :

### Option A — `<script type="text/plain">` multiples (le moins invasif)
Plusieurs blocs `<script id="part-xxx" type="text/plain">` avec une portion de JSX chacun, puis un script qui les concatène et appelle Babel sur le tout :
```js
const src = [...document.querySelectorAll('script[id^="part-"]')].map(s => s.textContent).join("\n");
Babel.transform(src, {...}).code;
```
- ✅ Pas de build, pas d'outil externe
- ✅ Tout reste dans `index.html`
- ❌ Démarrage encore plus lent (Babel-standalone reste lent)

### Option B — Build script PowerShell qui concatène (recommandé)
- Source dans `src/0_helpers.jsx`, `src/1_richtext.jsx`, `src/2_dashboard.jsx`, etc.
- Script `build.ps1` qui concatène + injecte dans `<script id="jsx-source">` de `index.html`
- Push de `index.html` final = même déploiement GitHub Pages
- ✅ Édition module par module, plus de scroll infini
- ✅ Babel ne voit qu'un seul fichier (perf identique)
- ❌ Étape supplémentaire avant push

### Option C — Build esbuild + JSX pré-compilé (le plus propre)
- Source en `.jsx` ES modules
- `esbuild` compile en un bundle JS minifié
- Supprime Babel-standalone du HTML (gain démarrage massif)
- ✅ Démarrage instantané
- ✅ Tree-shaking, type-check possible, tests possibles
- ❌ Devient un "vrai" projet Node — ce que l'app évite expressément aujourd'hui

**Pour l'instant** : on reste sur Option A implicite. Ce document sert de carte. À réviser quand `index.html` dépasse 1.5 Mo.
