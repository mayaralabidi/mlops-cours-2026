# Rapport comparatif — Kubeflow Pipelines vs Airflow (Lab 2)

**Binôme :** Mayara Labidi et Roua Khribiche · **Date :** 07/11/2026 · **Machine :** intel core i7, 12 Go de RAM , Windows 11 + WSL 2 (Ubuntu) ·
**Dataset :** german-credit

## 1. Contexte

Nous avons orchestré le pipeline de credit scoring du Lab 1 (german-credit, 1000 lignes, 20 variables, 30 % de mauvais payeurs) en six steps : ingest, validate, prepare, train, evaluate, register. Chaque step est un pod du cluster kind mlops-lab, lancé d'abord par Kubeflow Pipelines 2.17, puis par Airflow 3.3 (Docker Compose, KubernetesPodOperator). Les deux orchestrateurs partagent la même image projet (scoring-pipeline:1.0.1), le même code src/, le même MLflow et le même contrat dataset.yaml. Les résultats sont identiques : roc_auc 0,8094, seuil de décision 0,3182.

## 2. Résultats quantitatifs

| Mesure                                                  | KFP                                                 | Airflow                            |
| ------------------------------------------------------- | --------------------------------------------------- | ---------------------------------- |
| Installation de zéro (min)                              | ~15                                                 | ~10                                |
| Temps jusqu'au premier run réussi (min)                 | ~100 (pannes incluses)                              | ~20                                |
| Durée d'un run complet (min)                            | 4,7 (premier, avec tirage de l'image) ; 2,8 ensuite | ~2,3                               |
| Relance avec les mêmes paramètres (min)                 | ~2 (3 steps en cache)                               | ~2,3 (pas de cache)                |
| Hot-reload : modifier un step et voir le résultat (min) | ~4                                                  | ~3                                 |
| Lignes de code propres à l'orchestrateur                | 282                                                 | ~160 (70 de DAG + ~90 de stage.py) |
| Mémoire consommée par l'outil (Go, `docker stats`)      | ~4 (nœuds kind)                                     | ~2,2 (conteneurs Compose)          |

## 3. Analyse qualitative

### Developer experience

Airflow demande moins de code, mais il ne transporte aucun fichier : il a fallu écrire la logique de dossier de travail S3 dans stage.py. KFP demande plus de lignes, mais les artefacts typés, les métriques et la matrice de confusion s'affichent sans effort. Le cycle de modification diffère : Airflow relit le DAG seul en environ 30 s, alors que KFP impose de recompiler et d'uploader une version.

### Scaling

Dans les deux cas, un step est un pod : la capacité dépend du cluster, pas de l'outil. KFP apporte le cache et le parallélisme natifs, Airflow rejoue toujours toutes les tasks. Le gain du cache reste faible ici, car le pipeline est court et le jeu de données petit. Parallélisme : non testé (RAM limitée).

### Debugging

Environ 3 clics pour trouver l'erreur dans KFP (page du run, step rouge, onglet Logs) contre environ 4 dans Airflow (DAG, run, task rouge, Logs). Les logs KFP sont plus bruyants (lignes Argo), ceux d'Airflow ne montrent que la sortie du pod. Notre cas réel le plus formateur : après un arrêt brutal de Docker, l'image 1.0.0 était corrompue et les steps échouaient avec « accessing a corrupted shared library ». Seul un rebuild sous un nouveau tag (1.0.1) l'a corrigé, ce qui confirme la règle « image modifiée, nouveau tag ».

## 4. Recommandation

Nous choisirions KFP pour une équipe qui fait du ML sur Kubernetes et itère souvent sur l'entraînement : le cache, la traçabilité des artefacts et les visualisations font gagner du temps. Nous choisirions Airflow quand le pipeline ML n'est qu'un maillon de flux de données plus larges (bases, API, planification complexe), ou quand l'organisation l'utilise déjà. Dans les deux cas, il faut une image versionnée, des timeouts, des retries et un stockage fiable.

## 5. Limites de la comparaison

Tout a tourné sur un seul poste (kind à trois nœuds, 1000 lignes) : nous ne disons rien de la montée en charge réelle. Nous n'avons pas testé KFP sur un cluster managé (GKE), ni le KubernetesExecutor d'Airflow, ni l'authentification multi-utilisateur. Les durées sont des mesures uniques ou des estimations, perturbées par des incidents (port-forwards interrompus, image corrompue, UCI indisponible avec repli sur OpenML). La mémoire des nœuds kind inclut aussi MLflow et le stockage, donc elle surestime KFP seul.
