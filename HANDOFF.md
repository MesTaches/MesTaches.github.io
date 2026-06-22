# HANDOFF — Reprise de contexte « MesTaches »

> Document de reprise pour continuer le développement sur un autre compte/poste Claude.
> Colle ce fichier (ou son contenu) dans le 1ᵉʳ message d'une nouvelle conversation, ou garde-le dans le repo.
> **Lis aussi `CLAUDE.md` (architecture détaillée) et `MODULES.md` (carte des composants).**

---

## 1. Le projet en une phrase

Application web **mono-fichier** (`index.html`, ~1,2 Mo) de gestion de tâches + suivi financier (trésorerie / crédit clients), React 18 + Babel-standalone **compilé dans le navigateur**, déployée sur **GitHub Pages** (`https://mestaches.github.io`). Utilisateur : responsable trésorerie et crédit clients.

**Aucun build, aucun bundler, aucun test.** Le JSX vit dans `<script id="jsx-source" type="text/plain">` et est transformé par Babel au chargement.

## 2. Workflow de dev (impératif)

```powershell
# Éditer index.html, puis :
git add index.html
git commit -m "type(scope): description (v.0.NNN)"
git push origin main          # = déploiement GitHub Pages immédiat
```

- **Incrémenter la version à CHAQUE commit** : à 2 endroits → `<meta name="app-version">` (en tête) ET le footer sidebar (`v.0.NNN`). Version actuelle : **v.0.662**.
- Babel n'a pas de récupération d'erreur → une balise JSX mal fermée = **écran blanc total**. Tester (ouvrir l'app, F12 console) avant de pousser quand c'est risqué.
- Le repo Git EST la source de vérité portable du code (indépendant du compte Claude).

## 3. ⚠️ Pièges connus (déjà rencontrés — ne pas refaire)

- **Babel runtime** : le `Babel.transform` DOIT utiliser `presets: [["react", { runtime: "classic" }]]`. Une MAJ du CDN Babel (lien non versionné `unpkg.com/@babel/standalone`) avait basculé en runtime « automatic » qui émet des `import` ESM → écran blanc. **À sécuriser un jour : épingler la version de Babel.**
- **`isMobile is not defined`** : chaque composant qui utilise `isMobile` doit déclarer `const isMobile = useIsMobile();`. Plusieurs modules l'avaient oublié (écran blanc).
- **Écrasement par tableau vide** : un effet `useEffect([data])` qui sauvegarde APRÈS un chargement IDB qui a renvoyé `null` (transitoire, ex. pendant une montée de version IndexedDB) écrit `[]` par-dessus les données → **perte définitive**. Pattern de protection : un `skipNextSave` ref qui saute la 1ʳᵉ sauvegarde si le chargement n'a rien ramené (cf. EcheancierView). **Un emprunt a été perdu ainsi** (corrigé en v.0.660).
- **Onglet vide / écran blanc côté droit** : un nouvel onglet doit être ajouté à DEUX endroits — le ternaire `isFullWidthView` codé en dur (~ligne 17900) ET le bloc de rendu. (cf. Recouvrement, bug v.0.648.)
- **Apostrophes typographiques** dans les attributs JSX cassent Babel — utiliser des apostrophes ASCII `'`.
- **`typeof _auth`** : `_auth`/`_db` sont des `const` globales (script auth). Tester `typeof _auth !== "undefined"` (pas `window._auth`).

## 4. Stockage & synchronisation (CRUCIAL)

3 couches :
- **IndexedDB `app_main_db`** (helpers `appDbGet(store)`/`appDbSet(store,data)`) — stores : tasks, dev, contentieux, cotations, payments, echeancier, rcf, suivi_clients, suivi_clients_notes, **recouvrement**, meetings, settings, etc. Version actuelle **v4**.
- **IndexedDB `creditbail_db`** — séparé, pour les dossiers + PJ du crédit-bail.
- **localStorage** — pont legacy + déclencheur de la sync Firestore.

**Sync Firestore** : `appDbSet` écrit aussi dans localStorage pour les stores listés dans `STORE_TO_SYNC_KEY` → le monkey-patch `Storage.prototype.setItem` réplique vers Firestore (`users/{uid}/ls/{docKey}`). Stores synchronisés : tasks, settings, meetings, dev, contentieux, payments, cotations, suivi_clients(+notes), recouvrement.
- **NON synchronisés par ce pont** : `echeancier` (emprunts) et `creditbail` — trop volumineux / PJ.
- **Crédit-bail a sa PROPRE sync Firestore** (option B, v.0.656+) : métadonnées dans `users/{uid}/cb/meta`, PJ découpées en morceaux <1 Mo dans `users/{uid}/cbfiles/{cle}_{i}`, fusion par `_mtime`, pacing + retry backoff. Bouton de sync dans la sidebar CB.

## 5. Sauvegardes (3 niveaux) — Paramètres → 💼 Sauvegardes

1. **IndexedDB** local (temps réel).
2. **Snapshots Firestore** quotidiens (`users/{uid}/snapshots/{date}`), rotation 7 jours. Si payload >950 Ko, retire les plus gros stores un par un (ne jette plus l'échéancier — corrigé v.0.660).
3. **Fichiers JSON automatiques sur le PC** (v.0.661) via File System Access API : l'utilisateur choisit un dossier une fois, l'app y écrit `mestaches_backup_AAAA-MM-JJ.json` + `_latest.json`, automatiquement (quotidien/hebdo) + bouton manuel. Dossier actuel choisi : « Application Mes Taches ».

**Firebase** : projet `mes-taches-632e4`, forfait **Spark gratuit** (PAS de Storage — il exige Blaze/carte bancaire ; on a donc tout fait via Firestore). Auth email/mot de passe.

## 6. Modules / onglets

Tâches, Ma journée, Kanban, Catégorie, Réunions, Calendrier, Rappels, **Suivi clients**, **Recouvrement** (encours à risque), Contentieux, Cotation, Reporting, **Emprunts (échéancier)**, RCF, **Crédit-bail**, Paiements, Capex consolidé, Dev, Paramètres.

### Faits marquants récents
- **Recouvrement** (nouveau module, v.0.647+) : encours à risque clients. Gravité 🔴🟠🟢, montants/dépassement/ancienneté auto, type de garantie + montant garanti, graphique couverture (garanti vs non garanti), workflow d'action, échéances, notes. Transferts : Suivi clients → Recouvrement, et Recouvrement → Contentieux. Police Lora.
- **Crédit-bail** : sous-onglet **« Offres reçues »** (dossiers en « Demande de cotation », comparatif des offres financeurs, meilleur TEG surligné, total en haut à droite). PJ max 20 Mo. Sync Firestore (option B).
- **Emprunts** : `calcEcheancier()` (ligne ~458) génère un tableau d'amortissement depuis montant/taux/durée/**périodicité** (mensuel/trim/sem/annuel, v.0.662)/date/type amort/différé/in fine/assurance/garantie. 2 méthodes de création : import Excel OU saisie des paramètres + bouton « Générer l'échéancier ». Catégorisation des lignes (Versement, Disponibilité, Amortissement…) ; versements sans N°.
- **Paiements** : tableau en colonnes fractionnaires (pas de scroll horizontal), police Lora, montants alignés tabulaires.

## 7. État actuel / à faire

- ✅ Tout fonctionne en v.0.662. App stable.
- 🔁 **À ressaisir par l'utilisateur** : l'unique emprunt perdu (avant correctif v.0.660). Désormais protégé.
- 💡 **À valider** : la génération d'échéancier du prêt Capex SG (3 m€, probablement trimestriel) — comparer le tableau généré à celui du contrat PDF ; ajuster si écart (base de calcul Exact/360 vs 30/360, etc.).
- 💡 **Amélioration recommandée** : épingler la version de Babel (lien CDN versionné) pour éviter qu'une MAJ ne recasse l'app.

## 8. Conventions de code

- Polices : `_FL = "'Lora',serif"` (texte) ; `_DM = "'DM Mono',monospace"` (chiffres/petites caps). L'utilisateur préfère **Lora** dans les nouveaux écrans.
- Styles partagés : `S.btn(bg, color)`, `S.ghost`, `S.input`.
- Setter d'objet : `setD(p => ({...p, champ: val}))`.
- Helpers : `uid()`, `today()`, `fmtDate()`, `parseNum()`, `fmtEuro()`.
- Codes couleur : bleu marine `#102A63`, orange `#E67E22`/`#A16207` (crédit-bail/emprunts), vert `#065F46`/`#16A34A`, rouge `#C0392B`/`#B91C1C`.
- Pas de Fragment React (utiliser `<div>`).
- Suppressions : bandeau undo 10 s + corbeille 30 j selon les modules.

---

*Dernière mise à jour : v.0.662. Pour l'historique détaillé, voir `git log`.*
