# SalesDesk Pro — Contexte projet

## 🎯 Objectif du projet
SalesDesk Pro est un dashboard **single-file HTML** (~16 700 lignes au 31 oct. 2026) utilisé par Nicolas Lapeyre (BDR RealAdvisor France, qui mène et signe lui-même ses R1/R2) pour piloter sa journée : RDV du jour, suivi/relances, prospection, stats. Il **écrit dans Close** (statuts, tâches, notes, mails) via le copilot local ; Close reste la référence des faits commerciaux, SalesDesk organise le travail.

## 📂 Structure
- **Un seul fichier** : `index.html` (HTML + CSS + JS inline)
- **Hébergé** : GitHub Pages sur `niko130111.github.io/salesdesk-pro2`
- **Stockage** : Supabase (table `salesdesk_store`, ligne `main`, colonne jsonb `data`) = source partagée entre appareils, protégée par RLS + compte Auth (voir `docs-supabase-rls.md` du copilot) ; `localStorage` (`salesdesk_rdvs`) = copie locale. Gardes anti-écrasement : `_sdCloudVu`, `salesdesk_cloud_base`, `_sdChargementInitial`, PATCH conditionné à `data->>_lastWrite` (31 oct. 2026). **Synchro vivante** (31 oct. 2026) : événement `storage` entre onglets, veille cloud 20 s + retour sur l'onglet (`sdVeilleCloud`), une seule sauvegarde en vol.
- **Copilot** : serveur Node local `~/Desktop/Mes projets/sales-copilot/sales-copilot-v2/server/` (port 3015, `COPILOT_URL`) — toutes les lectures/écritures Close, Calendly, SMS, mails, RealAdvisor passent par lui.

## 🔑 Terminologie
- **R1** = premier rendez-vous avec le prospect
- **R2** = second rendez-vous (rescheduling ou suite du R1)
- **Outbound** = RDV pris via le lien Calendly personnel de Nicolas
- **Inbound** = RDV pris via le lien Calendly de l'équipe / inbound organique
- **Show** = présence effective au RDV
- **No Show / Annulé / Reprogrammé** = RDV non tenu
- **Mandataire / Agences** = 2 types de prospects (avec des taux commerciaux différents)

## ⚖️ Règles métier critiques (invariants)
1. **R2 ne pollue JAMAIS les stats R1**. Toujours filtrer `source !== 'R2'` avant tout calcul R1.
2. **Total RDV** = R1 avec présence renseignée uniquement (les futurs non traités ne comptent pas)
3. **Mandataires + Agences** doit toujours égaler le total R1
4. **Outbound** : 1 point par RDV (Mandataire ou Agences) — la règle « Agences = 2 pts » est terminée depuis le 4 oct. 2026
5. **Jours fériés français** : à exclure dans tous les calculs de "jours ouvrés"
6. **Deduplication Outbound** : par email, sur 2 mois glissants
7. **Matching auto R2 Calendly** : DÉSACTIVÉ (générait des doublons)

## 🗄️ Structure localStorage
```js
salesdesk_rdvs = {
  "2026-05": [ {client, date, dateR2, source, type, presence, presenceR2, result, pack, freq, ...}, ... ],
  "2026-06": [ ... ],
  _processedRdvs: { "nom|date": {...} },  // RDV masqués du Quotidien
  _prepRdvs: [...],
  _postRdvs: [...],
  _settings: {...},
  _manualRdvs: [...]
}

salesdesk_outbound_overrides = { ... }
salesdesk_outbound_types = { "email_prospect": "Mandataire"|"Agences" }
salesdesk_targets = { ca, rdv, shows, signes, showrate, closing, commission }
salesdesk_processed = { ... }  // ancien nom, legacy
```

## 🧩 Fonctions clés
- `getJoursOuvres(month, year)` — calcul jours ouvrés (exclut fériés FR 2025-2027)
- `getLast3FinishedMonthsStats()` — retourne stats des 3 derniers mois finis (comm avg, show rate, closing)
- `recalculateAutoTargets()` — remplit auto les champs Objectifs & Config depuis stats 3 mois
- `classifyRow(row)` — classification pour Pipeline (returns 'won'/'lost'/'noshow'/etc.)
- `dailyCalRenderDay()` — rend la vue Quotidien pour un jour donné
- `renderRdvTable()` — rend le tableau RDV (5 onglets : untreated/r1/r2/outbound/all)
- `sortRdvTable(field)` — tri par colonne
- `getComm(row)` — calcule la commission d'un RDV signé (barème complexe)
- `getAllProcessed()` — retourne les RDV marqués traités
- `markRdvAsProcessed(rdvId, data)` — marque un RDV comme traité (utilise nom|date, pas index)
- `persistCalendlyEventsToStore()` — sync Calendly → store (skip R2)

## 📊 Onglets (état au 31 oct. 2026)
- **Aujourd'hui** (`page-aujourdhui`, `renderAujourdhui`) — RDV du jour avec cartes Calendly, demain, priorités ; l'ancien « Quotidien » (`page-daily`) est redirigé dessus
- **Suivi** (`page-relances`, `renderRelancesPage`) — file de relances = tâches Close ouvertes + relances SalesDesk, deux panneaux, barre d'actions commune (Appeler, Close, RDV, Tâche, Modifier, Prioritaire, Plus)
- **Prospection** (`page-prospection`) — listes d'appel nocturnes du copilot (`/listes-appel`), secteurs, « Hors Close »
- **Copilot** (`page-mesrdv`) — le copilot intégré (prépa, live, après-RDV — Nicolas n'utilise pas l'après-RDV)
- **Pipeline** (`page-pipeline`) — Kanban drag & drop
- **Performance** (`page-analytics`) — KPIs mensuels, donuts, détail par source ; **Global** (`page-global`) vue annuelle
- **Bilan hebdo** (`page-bilan`), **Créneaux** (`page-creneaux`), **Contrôle** (`page-controle`), **Automatisations** (`page-automatisations`)
- **Données brutes** (`page-rdvtable`, `renderRdvTable`) — tableau éditable (5 onglets)
- **Objectifs & Config** (`page-settings`) — cibles, config, compte Supabase, sauvegardes
- **Fiche client** (`fc-overlay`, `ouvrirFicheClient`/`fcRendre`) — en-tête (Prochaine étape, ⏰ Tâche Close), onglets Synthèse / Échanges / Mails / Offre / Rendez-vous

## ✍️ Ce que SalesDesk écrit dans Close (via le copilot)
- Statut d'opportunité et de lead (`sdStatutClose` → `/close/statut` : show, no_show, annule, reprogramme, pas_interesse, ghost, mail_envoye, laisser_tomber) ; un échec affiche un toast rouge
- Tâches (`/close/taches`, règle **une tâche ouverte par client** : « Remplacer » par défaut après affichage de l'existante, sinon « Ajouter »), déplacement/clôture (`/close/taches/:id`), snooze du soir vers le prochain jour ouvré
- Notes (`/close/note`), mails (`/close/mail`, statut `outbox` ; validation obligatoire pour les mails automatiques), appels (`/close/appeler`, `/close/appel-sans-reponse`), escalade manager (`/close/escalader`), transmission (`/close/transmettre`), création de fiche au 1er appel « Hors Close » (`/close/lead/creer`), onboarding (`/close/onboarding*`)
- Homonymes : même nom + téléphone ou mail différents = deux personnes (`sdMemeContact`, 31 oct. 2026) ; un même prospect peut avoir plusieurs fiches Close

## 🎨 Préférences de code de Nicolas
- **Communication** : français, rapide, souvent abrégé
- **Actions directes** : préfère les modifs qu'on peut appliquer plutôt que des instructions
- **Stat coherence obsession** : toute incohérence entre 2 chiffres doit être fixée
- **Backup obligatoire** avant toute modif destructive du store : `localStorage.setItem('salesdesk_rdvs_backup_' + Date.now(), JSON.stringify(data))`
- **Suppression d'index** : toujours du plus grand au plus petit (`.splice(idx, 1)`)
- **Phrases motivation** : grandes (24px+), colorées, gras
- **UI** : soigné, aéré, cohérent avec le reste du dashboard

## 🛠️ Workflow typique
1. Nicolas décrit un besoin (souvent avec screenshot)
2. Tu identifies la ou les fonctions concernées
3. Tu modifies `index.html` (str_replace ciblé, pas de réécriture)
4. Tu valides la syntaxe JS (`node -e "..."` pour parser)
5. Tu commit + push sur GitHub
6. Il refresh son SalesDesk et valide visuellement

## 🚫 Ne PAS faire
- Ne pas réécrire des grosses sections du fichier
- Ne pas ajouter de dépendances externes (garder single-file HTML)
- Ne pas modifier les données de localStorage sans backup
- Ne pas casser la compat des mois passés (le store contient l'historique depuis oct 2025)
- Ne pas activer `autoMatchR2FromCalendly` (bug connu, doublons)
- Ne jamais déclencher une vraie écriture Close / Calendly / SMS / Supabase pendant un test : copie locale (`python3 -m http.server 8765`), `fetch` simulé, noms fictifs
- Ne pas remettre le verrou « un seul onglet écrit » (supprimé le 31 oct. 2026 au profit de la synchro vivante)

## 📅 Jours fériés français 2025-2027 (hardcodés dans `getJoursOuvres` et `_frenchHolidays`)
- 2025 : 01/01, 21/04, 01/05, 08/05, 29/05, 09/06, 14/07, 15/08, 01/11, 11/11, 25/12
- 2026 : 01/01, 06/04, 01/05, 08/05, 14/05, 25/05, 14/07, 15/08, 01/11, 11/11, 25/12
- 2027 : 01/01, 29/03, 01/05, 06/05, 08/05, 17/05, 14/07, 15/08, 01/11, 11/11, 25/12

## 🔗 Ressources
- Repo : `github.com/niko130111/salesdesk-pro2`
- Live : `niko130111.github.io/salesdesk-pro2`
- Calendly user URI : `https://api.calendly.com/users/f350a265-f5cb-45f5-a491-b3c864a2f7d3`
- Calendly outbound event type : `https://api.calendly.com/event_types/d6280962-6237-4f86-9d92-8320c817f400`
