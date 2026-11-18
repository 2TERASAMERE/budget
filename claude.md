# CLAUDE.md — monbudget (v2)

Application web de gestion de budget mensuel pour particulier. Fichier unique `index.html`, sans build. Backend **optionnel** : Supabase (auth email + mot de passe, synchronisation). Cible : usage navigateur aujourd'hui, **PWA / app mobile ensuite** (garder le mobile-first en tête à chaque modification).

## Stack
HTML / CSS / JS vanilla · `localStorage` · Chart.js 4.4.1 (cdnjs) · Font Awesome 6.5 (cdnjs) · Sora + DM Mono (Google Fonts).

## Fonctionnalités v2
- **Saisie** : catégories → sous-catégories → lignes (libellé, montant, marqueur récurrent ↻). Solde en temps réel.
- **Catégories personnalisables** : ajout / renommage / suppression, avec couleur, icône et type (`in` = revenu, `out` = dépense).
- **Sous-catégories** : ajout / renommage / suppression dans chaque catégorie (ex. Courantes → Alimentation, Transports, Santé, Divers).
- **Analyse** : donut des sorties par catégorie, récapitulatif, et **panneau de duplication** : liste les lignes marquées récurrentes (cochées par défaut), choix du mois cible (3 prochains mois), copie par fusion sur le libellé (met à jour le montant si la ligne existe déjà).
- **Historique** : solde des 6 derniers mois + graphique revenus vs sorties.
- Swipe gauche/droite = changement de mois (mobile).

## Modèle de données (localStorage)

| Clé | Contenu | Portée |
|---|---|---|
| `mb2_sections` | définition des catégories et sous-catégories | globale (tous les mois) |
| `mb2_{année}_{mois}_{secId}_{scId}` | tableau de lignes | par mois et sous-catégorie |

`mois` = `Date.getMonth()` (0–11).

```json
// mb2_sections
[{ "id":"fixes","name":"Dépenses fixes","color":"#ff4d6d","icon":"fa-house","type":"out",
   "subcats":[{ "id":"sc_fix_logement","name":"Logement" }] }]

// mb2_2026_9_fixes_sc_fix_logement
[{ "label":"Loyer / charges","amount":750,"recurring":true }]
```

- Structure des catégories = **globale** ; montants = **par mois**.
- Si aucune donnée n'existe pour un mois, `getRowsFor` retombe sur les lignes par défaut de `DEFAULT_SECTIONS` (montants à 0).
- Solde = Σ catégories `in` − Σ catégories `out`. Épargne et crédit sont des catégories `out` identifiées par leur id (`epargne`, `credit`) pour l'en-tête.

## Fonctions clés (appelées depuis le HTML inline — ne pas renommer)
`toggleSec` · `toggleSubcat` · `addRow` · `deleteRow` · `updateRowField` · `toggleRecurring` · `changeMonth` · `switchTab` · `openNewSectionModal` · `openEditSectionModal` · `deleteSection` · `openNewSubcatModal` · `openEditSubcatModal` · `deleteSubcat` · `executeDuplicate` · `selectColor` · `selectIcon` · `saveModal` · `saveSubcatModal`.

Accès données : `getSections` / `saveSections`, `getRows` (mois courant) / `getRowsFor(y,m,secId,scId)` / `saveRows`.

## Règles de développement
- **Ne jamais re-render une ligne pendant la saisie** (`updateRowField` met à jour les totaux sans reconstruire le DOM, sinon le champ perd le focus).
- Échapper toute chaîne injectée dans le HTML avec `esc()` (gère aussi les guillemets des attributs `value`).
- Inputs en `font-size ≥ 16px` sur mobile pour éviter le zoom iOS (fait, dans la media query `max-width:600px`).
- Pas de `float` brut à l'affichage : passer par `fmt()` (arrondi).
- Chart.js : donut en `responsive:false` (canvas créé dans un onglet masqué).
- Mobile (≤ 600px) : cibles tactiles ≥ 44px, safe-area (`viewport-fit=cover` + `env(safe-area-inset-*)`), lignes en grille à 2 niveaux (libellé pleine largeur, puis ↻ / montant / suppression). Les actions de catégorie et de sous-catégorie sont dans le corps repliable, pas dans l'en-tête.
- Swipe de mois ignoré dans les champs, modales et graphiques, et seulement si le geste est nettement horizontal.
- Après une synchro, jamais de re-render pendant la saisie : passer par `safeRender()` (reporté au `focusout`).

## Roadmap
1. **PWA** : `manifest.json`, `sw.js` (cache offline), icônes 192/512, `theme-color`, meta Apple. Héberger en HTTPS. Bundler Chart.js et Font Awesome en local pour le hors-ligne.
2. **Mobile** : barre d'onglets fixe en bas (cibles 44px, inputs 16px et safe-area déjà faits).
3. **Sauvegarde** : export / import JSON (localStorage = perte si cache vidé). Priorité haute.
4. **Migration v1 → v2** : anciennes clés `mb_{y}_{m}_{section}` non reprises (à écrire si des données v1 existent).
5. Budget cible par catégorie, objectif d'épargne, duplication vers un mois arbitraire, export PDF/CSV.
6. Natif si besoin : Capacitor (réutilise le code web) avant React Native / Flutter.

## Pièges connus
- Les données sont liées au navigateur et au domaine/fichier : changer de navigateur ou ouvrir le fichier depuis un autre chemin = données différentes.
- La duplication n'écrit que dans le mois cible choisi ; elle ne crée pas de catégories manquantes (la structure étant globale, ce cas n'existe pas).
- Supprimer une catégorie supprime ses données de **tous** les mois.

## Cloud — Supabase (optionnel)
- Config : `SUPABASE_URL` + `SUPABASE_ANON_KEY` en haut du bloc « CLOUD » de `index.html` (clé publique uniquement, **jamais `service_role`**). Vides = app 100 % locale.
- Schéma : `supabase_setup.sql` (`budget_config` = structure ; `budget_entries` = 1 ligne par mois × sous-catégorie, colonne `rows` jsonb, clé primaire `(user_id, year, month, section_id, subcat_id)`), RLS par utilisateur, `GRANT` explicites à `authenticated` uniquement. Ce schéma reflète 1:1 les clés localStorage (choisi pour une synchro simple et fiable).
- Synchro **offline-first** : toute écriture passe par `saveSections` / `putRows` → localStorage immédiat + file `mb2_pending` → `flush()` (upsert par clé primaire). Les suppressions passent par les ops `delSec` / `delSc`.
- `syncAll()` (connexion, retour en ligne, retour d'onglet) : flush des modifications en attente, puis le cloud fait foi pour la structure, et les lignes seulement locales sont envoyées (fusion). Conflits entre appareils : dernière écriture gagnante.
- Toute nouvelle écriture de données doit passer par `putRows` / `saveSections`, jamais par `localStorage.setItem` direct (sinon elle n'est pas synchronisée).
- Plan gratuit : projet mis en pause après ~1 semaine d'inactivité → l'app reste utilisable en local, la synchro reprend à la restauration.
- Auth : confirmation d'email désactivée côté Supabase ; fermer les inscriptions après création du compte.
- Clé à utiliser : **publishable** (`sb_publishable_…`) ; l'ancienne `anon` fonctionne encore mais est en cours de retrait. La librairie `supabase-js` est figée sur une version récente (≥ 2.110) via jsDelivr.
- Compte : créé à la main dans Authentication → Users (Auto Confirm User), puis inscriptions publiques désactivées. Le bouton « Créer un compte » de l'app ne sert plus dans ce cas.
