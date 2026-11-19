# Analyse statique TP4 — classement par règle

Base : 64 violations MISRA. Après corrections : 57.

| Règle | Avant | Après | Classe | Action | Commit |
|---|---|---|---|---|---|
| 13.3 | 1 | 0 | défaut réel | corrigé | 956fa3f |
| 12.3 | 3 | 0 | violation de norme | corrigé | bc1c00c |
| 12.1 | 3 | 0 | violation de norme | corrigé | bca2a62 |
| 15.5 | 25 | 25 | dérogation | DEV-004 | à faire |
| 17.7 | 9 | 9 | violation de norme (Required) | corriger | à faire |
| 5.9 | 4 | 4 | violation de norme | corriger (renommer) | à faire |
| 18.4 | 4 | 4 | dérogation | DEV-005 | à faire |
| 10.8 | 4 | 4 | défaut réel ou norme | corriger après vérification | à faire |
| 8.9 | 2 | 2 | violation de norme | corriger | à faire |
| 8.7 | 2 | 2 | faux positif | suppression ciblée | à faire |
| 21.6 | 2 | 2 | dérogation | DEV-002 | à faire |
| 7.4 | 1 | 1 | faux positif | cppcheck-suppress | à faire |
| 2.5 | 1 | 1 | à examiner | voir node.c (set_period) | à faire |
| 22.8 | 1 | 1 | défaut réel | errno = 0 avant strtol | à faire |
| 21.10 | 1 | 1 | dérogation | DEV-006 | à faire |
| 21.1 | 1 | 1 | dérogation | DEV-006 | à faire |
