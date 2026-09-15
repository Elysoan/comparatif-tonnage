# comparatif-tonnage

Outil de contrôle exutoire — recoupement mensuel des tonnages entre les
données de saisies internes (SILEX) et les données reçues de deux sites
exutoires (Sainte-Anne, Saint-Père). Objectif : détecter les écarts de
tonnage par flux, par jour et par tournée.

Application HTML standalone, sans serveur ni build : ouvrir
`controle_exutoire.html` dans un navigateur suffit.

## Fichiers sources

Trois fichiers Excel importés côté client (aucune donnée n'est envoyée à un
serveur) :

### SILEX — données saisies internes

| Donnée   | Colonne source          |
|----------|--------------------------|
| Date     | `Date intervention`      |
| Flux     | `Déchet Code`             |
| Tournée  | `Désignation`             |
| Tonnage  | `Qté à éliminer` (en tonnes) |

### Sainte-Anne — exutoire 1

| Donnée   | Colonne source            |
|----------|-----------------------------|
| Date     | `Date d'entrée`             |
| Flux     | `Matière annoncée`          |
| Tournée  | `Lieu d'exploitation`       |
| Tonnage  | `Poids de la matière` (en tonnes) |

### Saint-Père — exutoire 2

| Donnée   | Colonne source                  |
|----------|-----------------------------------|
| Date     | `Date du poids d'entrée`         |
| Flux     | `Libellé produit`                 |
| Tournée  | `Libellé tiers`                   |
| Tonnage  | `Net` (en kilogrammes → conversion ÷ 1000) |

## Architecture technique

HTML standalone, zéro dépendance serveur. Une seule lib externe : SheetJS
(`xlsx.full.min.js`) via CDN pour la lecture des `.xlsx`.

Flux de traitement côté client :

1. Lecture des 3 fichiers via `FileReader` + `XLSX.read()`
2. Parse + normalisation de chaque source (dates, flux, unités)
3. Agrégation double : tonnages et comptage de tickets
4. Rendu dynamique : onglets par flux, blocs dépliables par jour, datagrid
   par tournée

## Normalisation des flux

La fonction `normalizeFlux()` mappe les libellés hétérogènes des 3 sources
vers un nom unifié :

| Codes/libellés sources | Flux unifié |
|---|---|
| OM, ORDURES MENAGERES, ORDURES MENAGERES RESIDUELLES | OM |
| CS-EMR, EMR, EMBALLAGES EN MÉLANGES | CS-EMR |
| CS-JMR, PAPIER | PAPIER |
| BIODECHETS, BIODÉCHETS, BIODECHETS SPAN C3, BIODECH* | BIODÉCHETS |
| VERRE | VERRE |
| DD-BOUE, BOUES | BOUES |

La fonction parcourt la map et teste si la valeur normalisée contient la
clé (`.includes()`), ce qui absorbe les variantes mineures.

## Structure de données agrégées

Deux agrégats parallèles construits sur la même structure
`{ flux: { date: { tournee: valeur } } }` :

- `aggregate(records)` → somme des tonnages
- `aggregateTickets(records)` → comptage de lignes (tickets de pesée)

Appliqués séparément sur Sainte-Anne et Saint-Père. SILEX n'a pas de ticket
de pesée (c'est la référence saisie).

## Rendu

**Onglets** : un par flux détecté automatiquement (union des flux présents
dans les 3 fichiers).

**Barre KPI par flux (mensuel)** : tonnage SILEX · tonnage exutoires ·
écart absolu · écart % · jours avec écart · total tickets de pesée.

**Bloc par jour (dépliable)** :

- En-tête : date · tonnage SILEX · tonnage exutoires · écart coloré ·
  nombre de tickets (Ste-Anne: X / St-Père: Y)
- Détail (tableau) : une ligne par tournée avec SILEX (t) · Ste-Anne (t) ·
  St-Père (t) · Total exut. (t) · Tickets · Écart

**Code couleur écarts** :

- ✅ Vert : < 2 %
- ⚠️ Orange : 2–5 %
- ❌ Rouge : > 5 % ou source absente d'un côté

## Points d'extension identifiés

- La table `normalizeFlux` est facilement extensible pour de nouveaux flux.
- Les seuils d'écart (`SEUIL_OK = 0.02`, `SEUIL_WARN = 0.05`) sont des
  constantes en haut de script.
- Le recoupement par tournée est actuellement exact (matching string) —
  une tolérance ou un mapping tournée pourrait être ajouté si les
  libellés divergent entre SILEX et les exutoires.
- Pas d'export pour l'instant (CSV ou Excel des écarts envisageable).
