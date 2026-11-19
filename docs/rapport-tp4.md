# Rapport TP4 : revue, analyse statique, SonarQube et Git sur ThermoNode

## 1. Contexte et environnement

ThermoNode comprend un firmware en C99 (`firmware/`) et une passerelle en Python (`gateway/`). Le TP vise la revue de merge requests, l'analyse statique, la qualité du code et la gestion de branches Git.

| Élément | Version ou valeur |
|---|---|
| Système | Ubuntu 24.04.3 LTS, machine virtuelle Vagrant (hôte Windows) |
| Compilateur | gcc, flags `-std=c99 -Wall -Wextra -Werror -Wconversion` |
| Analyse statique C | cppcheck 2.13, addon MISRA C:2012 |
| Lint Python | pylint 3.3.1, fichier `gateway/.pylintrc` |
| Tests Python | pytest 8.3.3, pytest-cov 5.0.0 |
| Plateforme qualité | SonarQube Community Build 26.11.0.134175 (conteneur `sonarqube:community`) |
| Scanner | sonar-scanner-cli 8.4.0.16527 (conteneur officiel, réseau hôte) |

Méthode : chaque correction a été faite sur une branche dédiée, avec un commit par règle ou par regroupement de règles, et une vérification de non-régression avant chaque commit.

## 2. Revue de la merge request EF-04 (Partie 1)

La MR `fix/EF-04-hysteresis` a été examinée. Cinq constats ont été relevés :

1. Le seuil de sortie est codé en dur (`high_centi - 500`) et ignore la valeur configurée par `threshold_set`.
2. Un opérateur virgule (`ev = THR_CLEAR, above = 0u;`) rend l'affectation peu lisible et masque un état.
3. Les accolades d'un bloc conditionnel ont été retirées, ce qui crée un risque d'erreur à la maintenance.
4. Le test `test_EF04_configured_hysteresis_is_used` est désactivé : son appel `RUN_TEST` est commenté, donc la régression n'est pas couverte.
5. `strncpy(node_id, id, TN_ID_LEN)` ne place pas de terminateur nul si l'identifiant atteint la taille maximale.

Deux défauts de revue (DR-TP4-01 et DR-TP4-02) sont détaillés dans `docs/review-MR7.md`.

**Validation.** Des tests de validation indépendants (V1 à V6, `validation/`) ont été exécutés sur `main` puis sur la branche de la MR. Sur `main`, les six tests réussissent. Sur la branche de la MR, le test V1 échoue (une hystérésis configurée à 300 n'est pas appliquée) et la vérification du terminateur de `node_id` échoue.

**Décision.** La revue conclut à « Demander des changements » : la MR ne peut pas être fusionnée en l'état.

## 3. Analyse statique MISRA C:2012 (Partie 2)

Le nombre de violations visibles, relevé par cppcheck avec l'addon MISRA, est passé de **64 à 0**. Le détail par règle est tenu à jour dans `docs/analyse-statique-tp4.md`.

| Règle | Correction | Commit |
|---|---|---|
| 13.3 | Incrémentation séparée de l'affectation (`node.c`) | 956fa3f |
| 12.3 | Déclarations séparées dans `frame.c` | bc1c00c |
| 12.1 | Parenthèses explicites sur les comparaisons mixtes | bca2a62 |
| 17.7 | Valeur de retour de `memcpy`, `memset`, `strncpy` ignorée explicitement, `snprintf` vérifié | 42233fc, 0d6dd38 |
| 10.8 | Conversions explicites, `ptrdiff_t` pour les différences de pointeurs | 0a8ecf4 |
| 5.9 | Variables de `ringbuf.c` et `filter.c` renommées pour éviter le conflit avec des identifiants globaux | bfc68e6 |
| 8.9 | Variables statiques déplacées dans la fonction qui les utilise | 15999b5 |
| 2.5, 7.4 | Macro utilisée (`NODE_PERIOD_MAX_S`) ; faux positif 7.4 justifié par commentaire | 94fa149 |
| 22.8, 22.10 | Remplacement de `strtoul` par un analyseur hexadécimal (`parse_hex4`) | 38e5c64 |
| 8.7 | Fonctions `range_check` et `state_name` rendues `static` | f49b4e5 |

**Dérogations.** Quatre dérogations sont documentées dans `docs/misra-deviations.md` et `firmware/misra-suppressions.txt`. Elles concernent 15.5 (DEV-004), 18.4 (DEV-005), 21.6 (DEV-002), 2.5, 21.1 et 21.10 (DEV-006). Les dérogations DEV-002, DEV-004, DEV-005 et DEV-006 sont **en attente d'approbation** de l'enseignante. Aucune n'a été acceptée sans validation.

Le bilan final de l'analyse (commit 5f3416d) recense ces 64 violations initiales et leur traitement.

## 4. Analyse SonarQube (Partie 3)

**Mise en place.** La plateforme est lancée par `ci/platform.compose.yaml`. Le réseau externe `ci` a dû être créé, et la variable `vm.max_map_count` portée à 262144, comme l'exige le moteur Elasticsearch embarqué. Le projet `thermonode` est créé en local. Le token d'analyse est généré par l'interface et stocké hors du dépôt, dans un fichier réservé à l'utilisateur.

**Scan.** Le scan est exécuté avec sonar-scanner-cli dans un conteneur, sur le réseau hôte. Résultats :

- les 11 fichiers Python sont analysés par l'analyseur natif, avec la couverture et les tests fournis par `reports/` ;
- le rapport cppcheck est importé au format « generic issue » (`reports/cppcheck-sonar.json`) : aucune violation à importer, ce qui est cohérent avec la section 3 ;
- le format d'import est déprécié (champ `rules` absent) ; ce point est à corriger pour une version future de SonarQube.

**Limite.** L'édition Community n'embarque pas d'analyseur C natif. Les fichiers C sont donc exclus de l'analyse (`sonar.c.file.suffixes=-`) et seule l'importation cppcheck les concerne. Les options B et C du fichier `sonar-project.properties` (plugin communautaire, édition Developer) n'ont pas été retenues.

**Quality Gate.** [À COMPLÉTER : conditions de la porte TVD telles que définies à la section 3.6 de l'énoncé, valeurs seuils, et statut obtenu après le scan final.]

## 5. Qualité de la passerelle Python

Le score pylint de `gateway/gateway` est passé de **8,85 à 9,90 / 10**, avec les corrections de la branche `chore/pylint` (commit 2e6b06d). Les corrections portent sur les docstrings de classes, les arguments inutilisés des callbacks paho (préfixés `_`), les conversions de `split` et les instructions multiples.

**Un défaut fonctionnel est signalé et non corrigé.** Dans `handler.py`, la condition `if False and last is not None and f.seq <= last` désactive en permanence la détection de doublons. Ce n'est pas une question de style : l'activer change le comportement de la passerelle. Le point est consigné dans le code et doit être tranché avec l'enseignante. Il est désactivé pour pylint par une dérogation commentée.

**Tests.** Les tests unitaires de la passerelle réussissent. Les tests de fumée (`gateway/tests/smoke/`) ne sont pas unitaires : ils vérifient un déploiement et lisent `GATEWAY_URL` et `MQTT_HOST`. Sans passerelle à l'adresse configurée, ils échouent, y compris sur `main`. Ils sont donc exclus de l'exécution unitaire (`--ignore=gateway/tests/smoke`).

## 6. Git, revue croisée et versionnement (Partie 4)

[À COMPLÉTER après la publication des branches : URL du dépôt distant, liens des trois merge requests, revue croisée (reviewer et constats), résolution du conflit sur `THR_DEFAULT_HYST` (valeur conservée : 400, avec commentaire « hystérésis EF-04 »), tag `v0.2.0` et sortie de `git describe --tags`.]

**Versionnement.** La version `v0.2.0` est une version mineure (nouvelles fonctionnalités rétrocompatibles). Un changement du format de trame ICD serait une version majeure, car il casse la compatibilité.

## 7. Réponses aux questions Q1 à Q5

[À COMPLÉTER : reprendre les réponses rédigées pour les questions Q1 à Q5 du sujet.]

## 8. Dette technique et plan d'action

[À COMPLÉTER : dette restante (dérogations en attente, défaut R1727, format d'import déprécié, tests smoke à brancher sur la staging) avec priorité et responsable proposé.]

## 9. Références

- MISRA C:2012, règles citées aux sections 3 et 4.
- Documentation SonarQube Community Build, import de problèmes externes (format « generic issue »).
- Documentation pylint 3.3, messages et désactivations ciblées.

