# Projet 10 : mettre en place un pipeline d'orchestration des flux

Pipeline **Kestra** + **DuckDB** qui automatise chaque mois, pour le caviste BottleNeck :

- le nettoyage et la fusion des données ERP et du site web ;
- le calcul du chiffre d'affaires ;
- la segmentation des vins en premium et ordinaires.

## Contexte

**BottleNeck** est un marchand de vin haut de gamme qui vend à des entreprises et à des particuliers. Stéphane, data analyst, a réalisé une première analyse à la main :

1. **Nettoyage** : suppression des valeurs manquantes et des doublons dans les exports ERP et CMS ;
2. **Jointure** : réconciliation des deux fichiers via une table de correspondance ;
3. **Calcul du CA** : chiffre d'affaires par bouteille, puis chiffre d'affaires total ;
4. **Segmentation** : identification des vins premium par le z-score de leur prix.

La mission consiste à **industrialiser cette démarche**. Le pipeline doit être automatisé, testé à chaque étape et planifié. Il livre chaque mois aux équipes produit :

- le rapport de CA ;
- la liste des vins premium, pour Laure (comptes entreprises) ;
- la liste des vins ordinaires, pour Maria (comptes particuliers).

## Données sources

| Fichier              | Origine                 | Contenu                                                                         | Clé           |
| -------------------- | ----------------------- | ------------------------------------------------------------------------------- | -------------- |
| `data/erp.csv`     | ERP                     | `product_id`, `onsale_web`, `price`, `stock_quantity`, `stock_status` | `product_id` |
| `data/web.csv`     | CMS (site e-commerce)   | Fiches produits, titre, ventes (`total_sales`)…                              | `sku`        |
| `data/liaison.csv` | Table de correspondance | `product_id` ↔ `id_web`                                                    | `product_id` |

Les CSV ont été convertis depuis les fichiers Excel d'origine (`data/*.xlsx`). Ils sont publiés sur GitHub, et le workflow les télécharge à partir de leur URL brute (raw).

## Workflow Kestra

Le workflow est défini dans [kestra.yml](kestra.yml), avec l'identifiant `bottleneck_traitement` dans le namespace `bottleneck`.

```text
┌─ Parallel ─────────────────────────────────────────────┐
│ extract_erp → cleaning_erp → tests (manquants, doublons) │
│ extract_web → cleaning_web → tests (manquants, doublons) │
└──────────────────────────────────────────────────────────┘
extract_liaison → cleaning_liaison → tests (manquants, doublons)
        ↓
fusion (web ⟕ liaison ⟕ erp) → test de jointure
        ↓
calcul_CA → export_ca_excel (resultat_ca.xlsx) → test de cohérence du CA
        ↓
zscore (vin_premium.csv / vin_ordinnaire.csv) → tests du z-score et du nombre de vins premium
```

| Étape            | Traitement                                                                                      | Test (tâche `Assert`)                                             |
| ----------------- | ----------------------------------------------------------------------------------------------- | -------------------------------------------------------------------- |
| Nettoyage ERP     | Dédoublonnage sur `product_id`                                                               | 0 valeur manquante, 0 doublon                                        |
| Nettoyage web     | Seulement les lignes `post_type = 'product'` dont `sku` et `total_sales` sont renseignés | 0 valeur manquante, 0 doublon sur `sku`                            |
| Nettoyage liaison | Dédoublonnage sur `product_id`                                                               | 0 valeur manquante, 0 doublon                                        |
| Fusion            | `web.sku = liaison.id_web`, puis `liaison.product_id = erp.product_id`                      | Aucune ligne orpheline (`price` non nul)                           |
| Calcul du CA      | `ca = price × total_sales` par produit, et `ca_total`                                      | La somme des CA par produit est égale au CA total                   |
| Export            | Conversion du rapport en Excel (pandas, openpyxl)                                               |                                                                      |
| Segmentation      | z-score du prix : premium si `z ≥ 2`, ordinaire sinon                                        | Moyenne ≈ 0, écart-type ≈ 1, aucune valeur nulle, 30 vins premium |

### Planification et résilience

- **Déclenchement** : le 15 de chaque mois à 9 h (`cron: "0 9 15 * *"`).
- **Réessais** : 3 tentatives espacées de 30 secondes sur une tâche en échec, pour absorber une indisponibilité passagère de GitHub ou de DuckDB.
- **Arrêt en cas d'erreur** : si un `Assert` échoue, le flow s'arrête. Aucun résultat partiel ou faux n'est livré, et un message d'erreur explicite est écrit dans les logs Kestra.

### Livrables produits

| Fichier                | Contenu                                           | Destinataire                |
| ---------------------- | ------------------------------------------------- | --------------------------- |
| `resultat_ca.xlsx`   | CA par produit et CA total                        | Équipes produit            |
| `vin_premium.csv`    | Vins au prix statistiquement élevé (`z ≥ 2`) | Laure, comptes entreprises  |
| `vin_ordinnaire.csv` | Reste du catalogue                                | Maria, comptes particuliers |

## Lancement

Prérequis : Docker et Docker Compose.

```bash
# Kestra et sa base PostgreSQL
docker compose up -d
```

1. Ouvrez l'interface sur [http://localhost:8080](http://localhost:8080).
2. Créez un flow et collez le contenu de [kestra.yml](kestra.yml).
3. Lancez-le avec **Execute** ou attendez le déclenchement planifié.
4. Consultez les fichiers produits dans l'onglet **Outputs** de l'exécution.

Les vues **Gantt**, **Logs** et **Topology** de Kestra permettent de suivre la durée de chaque tâche, ses erreurs et l'enchaînement du flow.

## Structure du dossier

| Élément                                                               | Description                                                                   |
| ----------------------------------------------------------------------- | ----------------------------------------------------------------------------- |
| `kestra.yml`                                                          | Workflow Kestra complet                                                       |
| `docker-compose.yml`                                                  | Kestra (mode standalone) et PostgreSQL                                        |
| `data/`                                                               | Fichiers sources (Excel d'origine et CSV)                                     |
| `to_csv.ipynb`                                                        | Conversion des fichiers Excel en CSV                                          |
| `nettoyage.sql`, `dedoublonnage.sql`                                | Premières requêtes DuckDB de nettoyage, avant leur intégration dans Kestra |
| `identification.ipynb`, `indentification.py`                        | Prototype du calcul du z-score                                                |
| `vin_premium.csv`, `vin_ordinnaire.csv`                             | Résultats du prototype                                                       |
| `tache.md`                                                            | Journal de bord du projet                                                     |
| `Mettez_en_place_un_pipeline_d'orchestration_des_flux_siesse_thomas/` | Livrables : diagramme de flux (draw.io), workflow, présentation              |

## Pistes d'évolution

- Notifier automatiquement l'équipe technique (e-mail ou Slack) en cas d'échec.
- Envoyer automatiquement le rapport aux responsables produit.
- Historiser les CA mensuels pour suivre leur évolution.
- Appliquer la même méthode de tests aux autres flux de données de BottleNeck.

## Stack technique

Kestra, DuckDB, Python (pandas, openpyxl), PostgreSQL, Docker Compose, draw.io.

## Auteur

Thomas SIESSE, septembre 2026.
