# Tests de non-régression manuels

Liste à parcourir **avant chaque push** d'une modif significative. Babel-standalone n'a pas de récupération d'erreur — une `</div>` manquante = page blanche. La meilleure prévention reste un coup d'œil sur chaque module touché.

## ✅ Tests de fumée (5 min — à faire systématiquement)

À l'ouverture de `index.html` :

- [ ] **Page se charge** sans écran blanc (F12 → Console : aucune erreur rouge)
- [ ] Le **footer sidebar** affiche bien la nouvelle version `v.0.NNN`
- [ ] La **sidebar gauche** se déploie et permet de naviguer
- [ ] Cliquer sur **chaque onglet visible** dans la sidebar (Dashboard, Tâches, Catégories, Calendrier, Réunions, Cotation, RCF, Crédit-bail, Échéancier, Contentieux, Dev, Suivi clients, Notifications, Paramètres). Aucun ne doit déclencher un écran blanc.

Si écran blanc : F12 → Console → l'erreur exacte indique le composant fautif.

## 🎯 Tests ciblés par module

À déclencher uniquement quand on a modifié le module en question. Sinon ignorer.

### Tâches
- [ ] Création d'une tâche (`+ Nouvelle tâche`)
- [ ] Édition d'une tâche existante (clic dessus → modale latérale)
- [ ] Suppression d'une tâche + bandeau bleu "Annuler" pendant 10 s
- [ ] Filtre par catégorie / statut / priorité
- [ ] Recherche globale (Ctrl+K) trouve une tâche par titre

### Réunions
- [ ] Création (`+ Nouvelle réunion`)
- [ ] Sélection d'une réunion → la zone de texte se charge
- [ ] Bouton `+ Séance` insère un séparateur daté en haut
- [ ] Le `×` rouge sur un séparateur de séance le supprime
- [ ] Double-clic sur le titre d'une réunion dans la sidebar → renomme inline
- [ ] Double-clic sur une catégorie → renomme inline et propage à toutes les réunions de la catégorie
- [ ] Bouton 🗑 dans le header → confirmation "Supprimer ? Oui / Non" → suppression + undo 10 s
- [ ] Bouton 🗑 dans la sidebar header → ouvre la corbeille avec les réunions des 30 derniers jours
- [ ] Bouton `+ Tâche` → mini-modale pré-remplie depuis le texte sélectionné
- [ ] Bouton `📄 PDF` → nouvelle fenêtre avec impression auto

### Cotation
- [ ] Création d'une cotation
- [ ] Pagination (boutons Préc./Suiv. en bas du tableau)
- [ ] Filtres par statut / label

### Crédit-bail
- [ ] Ouverture d'un dossier → modale plein écran
- [ ] Upload d'une pièce jointe < 900 Ko (stockée inline)
- [ ] Upload d'une pièce jointe > 900 Ko (stockée dans `creditbail_db`)

### Échéancier
- [ ] Import CSV / XLSX

### Dashboard
- [ ] Panneau **Aujourd'hui** affiche les tâches en retard + du jour
- [ ] Clic sur une ligne → navigation vers le bon onglet

### Paramètres
- [ ] Saisie de la clé Groq → bouton IA apparaît dans les RichTextArea
- [ ] Toggle d'un onglet caché → l'onglet disparaît de la sidebar

## 🔄 Tests de sync (à faire de temps en temps)

- [ ] Modifier une tâche sur un appareil → vérifier qu'elle apparaît sur un autre dans les 10 s
- [ ] Se déconnecter / reconnecter Firebase → les données reviennent

## 🚨 Tests destructifs

Ne pas lancer en production. Sur une copie de `index.html` uniquement :

- [ ] `indexedDB.deleteDatabase("app_main_db")` + reload → la migration legacy se relance, aucune perte de données critique
- [ ] Supprimer le localStorage → l'app démarre toujours (signe-in Firebase requis)

## Workflow recommandé

```powershell
# 1) Modifier index.html
# 2) Incrémenter v.0.NNN dans <meta app-version> ET le footer sidebar
# 3) Tester localement : start index.html
# 4) Parcourir les Tests de fumée
# 5) Parcourir les Tests ciblés du module modifié
# 6) Si OK : commit + push
git add index.html
git commit -m "feat(<module>): <description> (v.0.NNN)"
git push origin main
```

## Quand un écran blanc apparaît

1. F12 → onglet Console
2. Lire la **première** erreur rouge (les suivantes sont des cascades)
3. L'erreur indique le composant fautif (`at CotationView`, `at MeetingsView`…)
4. Erreurs courantes :
   - `Cannot access 'X' before initialization` → un `useState` est référencé par un `useEffect` placé **avant** sa déclaration. Déplacer le `useEffect` après.
   - `X is not defined` → variable utilisée mais jamais déclarée (typique : `totalPages` sans pagination calculée).
   - `Unexpected token` → erreur de syntaxe JSX (balise mal fermée, apostrophe typographique `'` au lieu de `'`).
