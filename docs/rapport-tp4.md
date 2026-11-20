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

## 4. Quality Gate SonarQube

Analyse lancee sur `main` (commit 3c77188) : scan reussi (EXECUTION SUCCESS). La gate **TVD** est assignee au projet `thermonode` (verifie par l'API `get_by_project`).

Conditions de la gate (section 3.6) : couverture du nouveau code < 80 %, bugs du nouveau code > 0, vulnerabilites du nouveau code > 0, hotspots revus du nouveau code < 100 %, duplication du nouveau code > 3 %, maintenabilite du nouveau code > 1 (note A).

Resultat : statut **OK**, toutes les conditions evaluees sont OK. Reserve : `ignoredConditions` est vrai, donc certaines conditions sur le nouveau code n'ont pas ete evaluees, faute de periode de reference exploitable. Le statut OK doit etre lu avec cette reserve.

Metriques du projet entier : 196 lignes de code ; 0 bug ; 0 vulnerabilite ; 0 hotspot ; 1 code smell ; duplication 0,0 % ; couverture 0,0 % (le firmware C n'est pas mesure par SonarQube Community, voir Q3) ; notes de fiabilite, de securite et de maintenabilite a A.

## 5. Qualité de la passerelle Python

Le score pylint de `gateway/gateway` est passé de **8,85 à 9,90 / 10**, avec les corrections de la branche `chore/pylint` (commit 2e6b06d). Les corrections portent sur les docstrings de classes, les arguments inutilisés des callbacks paho (préfixés `_`), les conversions de `split` et les instructions multiples.

**Un défaut fonctionnel est signalé et non corrigé.** Dans `handler.py`, la condition `if False and last is not None and f.seq <= last` désactive en permanence la détection de doublons. Ce n'est pas une question de style : l'activer change le comportement de la passerelle. Le point est consigné dans le code et doit être tranché avec l'enseignante. Il est désactivé pour pylint par une dérogation commentée.

**Tests.** Les tests unitaires de la passerelle réussissent. Les tests de fumée (`gateway/tests/smoke/`) ne sont pas unitaires : ils vérifient un déploiement et lisent `GATEWAY_URL` et `MQTT_HOST`. Sans passerelle à l'adresse configurée, ils échouent, y compris sur `main`. Ils sont donc exclus de l'exécution unitaire (`--ignore=gateway/tests/smoke`).

## 6. Historique Git et livraison

Depot : fadimechi2001-commits/thermonode-tp4. La revue de la MR EF-04 est dans `docs/review-MR7.md` ; decision : Demander des changements.

Pull requests :
- #1 et #2 : corrections MISRA et pylint.
- #3 : `docs/deviations` ; conflit resolu en alignant le fichier sur `main`.
- #4 : `feat/threshold-defaults-doc`, seuil d'hysteresis a 400 (conflit simule).
- #5 : `docs/rapport-tp4`, rapport et reponses Q1 a Q5.
- #6 : `docs/git-history`, ajout de `docs/git-history.txt`.

Conflit simule : rebase de `feat/threshold-defaults-doc` sur `fix/threshold-300`. Conflit sur `THR_DEFAULT_HYST` dans `firmware/src/threshold.h`, resolu en gardant 400 (valeur retour terrain EF-04), puis `git add` et `git rebase --continue`.

Tag : `v0.2.0` pose sur `main` (commit 3c77188) et pousse ; `git describe --tags` retourne `v0.2.0`.

## 7. Réponses aux questions Q1 à Q5

**Q1 : techniques ayant révélé les cinq problèmes de la MR EF-04.**
- Seuil de sortie codé en dur (500 au lieu de 300) : révélé par un test à la main (`threshold_set(6000, 300)` puis `threshold_step(5699, true)`, qui doit donner CLEAR). La lecture seule ne suffit pas : c'est une technique dynamique.
- Opérateur virgule (`ev = THR_CLEAR, above = 0u;`) : révélé par la lecture avec la liste de contrôle. Il n'est pas remonté par cppcheck dans la configuration utilisée.
- Accolades retirées : révélées par cppcheck (règle MISRA 15.6). Analyse statique.
- Test désactivé (`RUN_TEST` commenté) : révélé par la lecture du diff, en vérifiant le périmètre des tests. La CI reste verte, donc l'outil ne le voit pas.
- `strncpy` sans terminateur : révélé par la lecture (chaînes et tampons) et confirmé par un test de validation (`node_id` sans terminateur).

Complémentarité : l'analyse statique voit la forme du code (accolades, parenthèses, règles), l'exécution voit le comportement (seuil de 500 au lieu de 300). Une suite de tests verte n'est donc pas un certificat : elle ne voyait ni le seuil, ni le test désactivé.

**Q2 : classement des 18 messages pylint.**
- Défaut réel : R1727 (`handler.py`, détection de doublons désactivée par un `False` en dur). Non corrigé : décision à prendre avec l'enseignante.
- Violations de norme corrigées : C0115 (docstrings, 5 occurrences), W0613 (arguments inutilisés des callbacks, 5 occurrences, renommés avec un préfixe `_`), C0207 (`maxsplit`), C0321 (instructions multiples).
- Faux positif lié à l'environnement : E0401 (`paho-mqtt` absent de la VM), résolu par l'installation de `requirements.txt`, sans changement de code.
- Dérogations justifiées, désactivées localement : C0103 (`do_GET`, nom imposé par `http.server`), R0913 (2 occurrences, signatures imposées ou justifiées), W0511 (TODO conservé et documenté).

Pourquoi une désactivation locale est acceptable : elle ne porte que sur une ligne ou une fonction, avec une justification écrite à côté, visible en revue. Une désactivation globale dans `.pylintrc` masque la règle pour tout le projet, sans motif. L'équivalent MISRA est la suppression ciblée dans `firmware/misra-suppressions.txt`, justifiée par une entrée DEV-xxx dans `docs/misra-deviations.md`.

**Q3 : asymétrie de couverture entre passerelle et firmware.**
La passerelle Python est mesurée par Sonar (rapport `coverage-gw.xml`). Le firmware C a une couverture produite par la cible `coverage` du Makefile (gcovr), mais Sonar Community ne dispose pas d'analyseur C : les fichiers C sont exclus (`sonar.c.file.suffixes=-`), et la couverture C n'est pas importée. La porte de couverture du Quality Gate ne voit donc que la passerelle. C'est acceptable dans cette édition, à condition de le dire : le pipeline du TP5, qui exécute la cible `coverage`, vérifie le firmware. Pour que Sonar voie le firmware, il faudrait l'option B (plugin communautaire) ou l'édition Developer.

**Q4 : `--force-with-lease` plutôt que `--force`, et tests verts malgré le changement de constante.**
`--force-with-lease` refuse le push si la branche distante a changé depuis notre dernier fetch : on n'écrase que ce que l'on a vu. `--force` effacerait en silence un commit ajouté par un autre membre. Les tests passent encore parce que la valeur par défaut n'est pas testée : d'après la lecture, `setUp` de `test_threshold.c` passe des valeurs explicites, et `node_init` dans `test_node.c` ne vérifie pas l'hystérésis par défaut. Le test manquant est : « `threshold_init` applique `THR_DEFAULT_HYST` ». (À confirmer dans les fichiers avant de rendre le rapport.)

**Q5 : versionnement sémantique.**
`v0.2.0` est une version mineure : elle ajoute des fonctionnalités (configuration de l'hystérésis, outillage qualité) sans rupture de compatibilité. La correction d'un seul défaut sans changement d'interface relève d'une version corrective, `0.1.1`. Le jour où l'ICD change le format de trame, la compatibilité avec les nœuds et la passerelle existants est rompue : c'est une version majeure, `1.0.0`.

## 8. Dette technique restante

- Couverture du firmware non mesuree : SonarQube Community ne lit pas les resultats Unity du firmware, d'ou une couverture globale affichee a 0,0 %. Action : integrer gcov/lcov dans un rapport externe.
- Defaut R1727 dans `gateway/gateway/handler.py` (condition constante) : la detection de doublon est desactivee par `pylint: disable`. A corriger.
- Derogations MISRA DEV-002, DEV-004, DEV-005 et DEV-006 : a valider par l'enseignant, et a reduire a terme.
- Tests smoke de la gateway (`gateway/tests/smoke`) exclus : ils necessitent `GATEWAY_URL` et `MQTT_HOST` de staging.
- Un code smell residuel ; conditions du nouveau code ignorees par la Quality Gate.

## 9. Références

- MISRA C:2012, règles citées aux sections 3 et 4.
- Documentation SonarQube Community Build, import de problèmes externes (format « generic issue »).
- Documentation pylint 3.3, messages et désactivations ciblées.

