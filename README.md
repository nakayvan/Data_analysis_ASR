 Analyse de suivi de programme : ASR (RESILIENCE+)

> Diagnostic et analyse des données du programme **RESILIENCE+** de l'ONG fictive **ASR (Action Sahel pour la Résilience)**, réalisés sous **Power Query, Power BI et DAX**.
> Cas pratique réalisé dans le cadre du Test 2 « Commande client » de **NP Training & Consulting**.

---

## Contexte

Action Sahel pour la Résilience (ASR) est une organisation non gouvernementale fictive intervenant dans plusieurs communautés d'Afrique francophone, dans le domaine de la sécurité alimentaire et des moyens d'existence. Avec le programme RESILIENCE+, elle accompagne des ménages vulnérables et de petits producteurs : accompagnement, formations, appui aux activités génératrices de revenus (AGR) et suivi des bénéficiaires.

Après plusieurs mois de mise en œuvre, l'équipe MEAL constate que les données collectées sont difficiles à exploiter rapidement pour le suivi du programme et la préparation des rapports. Sa responsable fait appel à NP Training & Consulting pour obtenir un premier diagnostic et une analyse des résultats.

## Objectifs

- **Suivi du programme** : qu'est-ce qui a été réalisé, pour qui et où ?
- **Tendances** : comment le programme évolue-t-il dans le temps ?
- Étudier la structure des données, évaluer leur fiabilité, appliquer des corrections utiles **uniquement lorsque c'est possible**, puis proposer une analyse fondée sur ce nettoyage.

---

## Données

Cinq jeux de données au format CSV, importés dans Power Query :

| Table | Rôle | Contenu principal |
|---|---|---|
| `activites` | Table de faits | 6 500 activités : date, type, modalité, thème, présence, durée en minutes, agent, zone |
| `beneficiaires` | Dimension | Sexe, âge, taille du ménage, vulnérabilité, statut, activité principale, date d'enregistrement |
| `zones` | Dimension | Commune, département, région, type de zone (20 zones) |
| `suivi_resultats` | Table de faits | Indicateurs mesurés par bénéficiaire : valeur, unité, source, date de suivi (3 200 suivis) |
| `dictionnaire_donnees` | Référence | Description des colonnes |

**Repères issus du profilage :**

- 6 500 activités réalisées, concernant 1 756 bénéficiaires distincts
- 5 types d'activité (Accompagnement, Formation, Sensibilisation, Distribution, Appui AGR) et 19 thèmes
- 40 agents
- 465 dates distinctes, du 14 janvier 2025 au 5 mai 2026

La colonne `zone_id` présente dans `activites` permet d'analyser la zone comme **le lieu de l'activité**, et pas seulement comme la zone de résidence du bénéficiaire.

---

## Démarche

1. **Comprendre la structure des données** : profilage de chaque table dans Power Query (valeurs distinctes, valeurs nulles, distribution)
2. **Traiter les problèmes de qualité** : corrections appliquées dans Power Query, jamais dans les fichiers sources
3. **Construire le modèle de données** : schéma en étoile, table de dates, table des agents
4. **Définir les indicateurs de suivi** : mesures DAX regroupées dans une table `_Mesure`
5. **Repérer les tendances** : lecture au mois, à partir de la table de dates
6. **Produire le reporting** : rapport Power BI organisé en pages

### Problèmes de qualité et corrections

Un compteur de valeurs distinctes supérieur à la valeur attendue sert de premier signal d'alerte (par exemple, 4 valeurs dans `modalite` au lieu de 3).

| Problème constaté | Colonne | Correction appliquée |
|---|---|---|
| « Presentiel » et « Présentiel » traités comme deux valeurs | `activites[modalite]` | Suppression des espaces, mise en forme de la casse, remplacement par « Présentiel » |
| Valeurs « Oui » et « Present » au lieu de « Présent » | `activites[presence]` | Remplacement par « Présent » (restent : Présent et Absent) |
| Sexe saisi sous plusieurs formes (« Femme », « Feminin ») | `beneficiaires[sexe]` | Harmonisation en « F » |
| Doublons sur l'identifiant (ex. B00101, B00401) | `beneficiaires[beneficiaire_id]` | Suppression des doublons, pour permettre une relation un-à-plusieurs |
| Espaces parasites dans les textes | `type_activite`, `theme` | Nettoyage par `Text.Trim` |
| Formats de date hétérogènes | `date_enregistrement`, `date_suivi`, `date_activite` | Passage au format AAAA-MM-JJ, conforme au dictionnaire de données |
| Valeurs manquantes (`Blank`) | `theme`, `activites_principales` | *[à compléter : décision prise et justification]* |
| Valeurs `NULL` | `age`, `valeur` | *[à compléter : décision prise et justification]* |

**Impact :** les valeurs nulles ou vides et les doublons faussent le suivi des performances et empêchent d'établir les liens entre les tables. C'est l'une des raisons pour lesquelles l'équipe MEAL était confuse devant les résultats du suivi.

**Vérification :** après nettoyage, le profilage confirme 3 valeurs dans `modalite` (Présentiel, Communautaire, À Distance) et 2 valeurs dans `presence` (Présent, Absent).

---

## Modèle de données

Modèle en étoile : les dimensions (`temps`, `zones`, `Agents`) filtrent les tables de faits. Toutes les relations sont en **un-à-plusieurs**, avec un filtre dans un seul sens.

| Côté « un » | Côté « plusieurs » | Colonnes | État |
|---|---|---|---|
| `temps` | `activites` | Date → `date_activite` | Active |
| `temps` | `beneficiaires` | Date → `date_enregistrement` | Inactive |
| `temps` | `suivi_resultats` | Date → `date_suivi` | Inactive |
| `zones` | `activites` | `zone_id` → `zone_id` | Active |
| `Agents` | `activites` | `agent_id` → `agent_id` | Active |

- **Table de dates `temps`** : créée dans Power Query, granularité journalière, continue (sans trou), avec année, trimestre, mois, clé Année-Mois et semaine. La lecture des tendances se fait au mois, car 465 dates distinctes seraient trop bruitées.
- **Relations inactives** : une seule relation peut être active entre deux tables, donc `date_enregistrement` et `date_suivi` sont activées dans les mesures avec `USERELATIONSHIP`.
- **Table `Agents`** : inexistante à l'origine, construite en réunissant les `agent_id` distincts de `activites` et de `suivi_resultats`.

---

## Indicateurs de suivi

| Indicateur | Question à laquelle il répond | Mesure |
|---|---|---|
| Nombre d'activités | Combien d'activités ont eu lieu ? | `COUNTROWS(activites)` |
| Bénéficiaires touchés | Combien de personnes distinctes ont participé ? | `DISTINCTCOUNT(activites[beneficiaire_id])` |
| Taux de présence | Quelle part des activités a eu lieu avec une personne présente ? | Nb présents / Nb activités |
| Heures totales | Quel temps a été consacré aux bénéficiaires ? | Somme de `duree_minutes` / 60 |
| Répartition par type d'activité | Où le programme investit-il son effort ? | Part de chaque type dans le total |
| Répartition par zone | Où le programme agit-il ? | Nb activités par région |

Exemple de mesures DAX :

```dax
Nb présents = CALCULATE([Nb activités], activites[presence] = "Présent")

Taux de présence = DIVIDE([Nb présents], [Nb activités])
```

**Point de contrôle :** les cartes du rapport affichent **6 500 activités** et **1 756 bénéficiaires touchés**, soit exactement les chiffres du profilage initial, ce qui confirme que les relations et le nettoyage ne déforment pas les données. Le **taux de présence ressort à 91,65 %**.

---

## Tendances

Les tendances utilisent le mois comme axe (clé Année-Mois).

| Tendance | Mesure |
|---|---|
| Évolution du volume d'activités | Nb activités |
| Évolution de la portée | Bénéficiaires touchés |
| Évolution de l'engagement | Taux de présence |
| Rythme de recrutement | Nouveaux bénéficiaires (via `date_enregistrement`) |
| Évolution des résultats | Valeur moyenne de l'indicateur, un indicateur à la fois |

**Premier constat :** le nombre d'activités monte de janvier 2025 jusqu'à un **pic en septembre-octobre 2025 (environ 750 activités par mois)**, puis baisse régulièrement jusqu'au début de 2026.

**Précautions de lecture :**

- Les données commencent le 14 janvier 2025 et s'arrêtent le 5 mai 2026 : janvier 2025 et mai 2026 sont des **mois partiels**, et leurs valeurs basses ne signalent pas un effondrement.
- La baisse entre octobre 2025 et avril 2026 est réelle dans les données, mais celles-ci n'en donnent pas la cause (fin de cycle, saisonnalité, retard de saisie). Elle doit être vérifiée auprès du responsable du programme.
- Les indicateurs de `suivi_resultats` ont des unités différentes : ils ne se moyennent jamais ensemble.

*[À compléter : constats sur la présence par mois, les nouveaux bénéficiaires et les résultats, une fois les courbes créées.]*

---

## Reporting

Le rapport Power BI est organisé en pages, chacune répondant à une question :

| Page | Contenu | Visuels |
|---|---|---|
| Vue d'ensemble | Le programme en chiffres | Cartes (activités, bénéficiaires, taux de présence), histogramme par type d'activité, courbe mensuelle |
| Tendances | L'évolution dans le temps | Courbes mensuelles, avec un segment sur l'indicateur |
| Caractéristiques des personnes | Le profil des bénéficiaires | Sexe, tranches d'âge, vulnérabilité, taille des ménages |

Règles de présentation : un message par visuel, des courbes triées dans l'ordre chronologique, et des segments (année, type d'activité, région) qui filtrent toute la page.

### Aperçu

*[À insérer : captures d'écran des trois pages du rapport, par exemple `![Vue d'ensemble](images/vue-ensemble.png)`.]*

---

## Difficultés rencontrées et solutions

| Difficulté | Cause | Solution |
|---|---|---|
| Erreur « cyclic reference » sur `Agents` | La requête se référençait elle-même | La requête pointe désormais vers `activites` et `suivi_resultats` |
| Erreur « import matches no exports » sur `activites` | Deux étapes collées dans la même ligne de code | Une seule étape de nettoyage, avec un nom unique |
| Chemins ambigus entre `activites` et `Agents` | Mauvaise colonne, cardinalité un-à-un, filtre bidirectionnel | Relation un-à-plusieurs, filtre dans un seul sens, sur `agent_id` |
| Courbe mensuelle trompeuse | Axe trié par valeur et non par date | Tri de l'axe Année-Mois par ordre croissant |
| Mesures refusées par Power BI | Plusieurs formules collées dans une seule expression | Une mesure par formule, dans la table `_Mesure` |

Ces erreurs ont appris à remonter d'un message d'erreur à l'étape fautive, et à vérifier chaque modification contre un chiffre de contrôle.

---

## Limites et perspectives

**Limites**

- Deux mois partiels (janvier 2025 et mai 2026) dans les tendances
- Les causes des évolutions ne peuvent pas être déduites des données seules
- Les indicateurs de résultats ne sont pas comparables entre eux
- `date_suivi` et `date_enregistrement` passent par des relations inactives et demandent des mesures dédiées
- Données fictives : les conclusions ne reflètent pas une situation réelle

**Perspectives**

- Analyser l'effet de l'intensité de participation sur les résultats
- Ajouter un suivi par agent pour mesurer la charge de travail
- Automatiser l'actualisation des données à chaque nouvel export

---

## Livrables

| Livrable | Fichier |
|---|---|
| Fichier de travail Power BI | `TEST2.pbix` |
| Restitution PowerPoint | `ASR__Guide.pptx` (« ASR : Guide de compréhension ») |
| Processus de travail (Livrable 3) | `Livrable_3_Processus_de_travail.docx` |

## Outils

- **Power Query** : import, profilage et nettoyage
- **Power BI Desktop** : modèle de données, mesures DAX et visuels
- **DAX** : calcul des indicateurs
- **Claude (IA)** : cadrage de la méthode à suivre, check-list des opérations et aide au débogage du code DAX et des relations entre tables. Les choix d'analyse et les décisions de traitement des données restent les miens.

## Structure du dépôt

```
asr-resilience-plus/
├── data/                  # Tables sources (CSV)
├── powerbi/
│   └── TEST2.pbix
├── docs/
│   ├── ASR__Guide.pptx
│   └── Livrable_3_Processus_de_travail.docx
├── images/                # Captures du rapport
└── README.md
```

---

## Auteur

**Tangou Ntchanga Yvan**
Étudiant en Master 1 Systèmes Embarqués et Intelligence Artificielle, Université de Douala (Cameroun)
Passionné par l'analyse de données, l'IA et le design de marque.

[LinkedIn](https://www.linkedin.com/in/tangou-yvan-87576029a)
