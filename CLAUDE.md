# SalesDesk Pro — Contexte projet

## 🎯 Objectif du projet
SalesDesk Pro est un dashboard **single-file HTML** (~9600 lignes) utilisé par Nicolas Lapeyre (BDR RealAdvisor France) pour tracker son activité sales indépendamment du CRM d'équipe.

## 📂 Structure
- **Un seul fichier** : `index.html` (HTML + CSS + JS inline)
- **Hébergé** : GitHub Pages sur `niko130111.github.io/salesdesk-pro2`
- **Stockage** : `localStorage` uniquement (clé principale `salesdesk_rdvs`) + Supabase (backup)

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

## 📊 Sections principales
- **Quotidien** (`page-daily`) — vue jour, blocs de pilotage, Calendly du jour
- **Analytics** (`page-analytics`) — KPIs mensuels, donuts, détail par source
- **Global** (`page-global`) — vue annuelle
- **Pipeline** (`page-pipeline`) — Kanban (drag & drop) : Won/Lost/No Show/Interesting
- **RDV Table** (`page-rdvtable`) — tableau éditable (5 onglets)
- **Objectifs & Config** (`page-config`) — cibles + config

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

## 📅 Jours fériés français 2025-2027 (hardcodés dans `getJoursOuvres` et `_frenchHolidays`)
- 2025 : 01/01, 21/04, 01/05, 08/05, 29/05, 09/06, 14/07, 15/08, 01/11, 11/11, 25/12
- 2026 : 01/01, 06/04, 01/05, 08/05, 14/05, 25/05, 14/07, 15/08, 01/11, 11/11, 25/12
- 2027 : 01/01, 29/03, 01/05, 06/05, 08/05, 17/05, 14/07, 15/08, 01/11, 11/11, 25/12

## 🔗 Ressources
- Repo : `github.com/niko130111/salesdesk-pro2`
- Live : `niko130111.github.io/salesdesk-pro2`
- Calendly user URI : `https://api.calendly.com/users/f350a265-f5cb-45f5-a491-b3c864a2f7d3`
- Calendly outbound event type : `https://api.calendly.com/event_types/d6280962-6237-4f86-9d92-8320c817f400`
