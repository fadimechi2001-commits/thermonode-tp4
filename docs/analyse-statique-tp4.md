# Analyse statique TP4 — classement par règle

Base : 64 violations MISRA. Maintenant : 10 visibles, plus 32 dérogées (DEV-004 et DEV-005), donc masquées par les suppressions ciblées.

| Règle | Avant | Après | Classe | Action | Commit |
|---|---|---|---|---|---|
| 13.3 | 1 | 0 | défaut réel | corrigé | 956fa3f |
| 12.3 | 3 | 0 | violation de norme | corrigé | bc1c00c |
| 12.1 | 3 | 0 | violation de norme | corrigé | bca2a62 |
| 15.5 | 25 | 27 (dérogées) | dérogation | DEV-004 | 1177a26 |
| 17.7 | 9 | 0 | violation de norme (Required) | corrigé | 42233fc, 0d6dd38 |
| 5.9 | 4 | 0 | violation de norme | renommage rb_ / flt_ | bfc68e6 |
| 18.4 | 4 | 5 (dérogées) | dérogation | DEV-005 | 1177a26 |
| 10.8 | 4 | 0 | violation de norme (Required) | casts sur variables simples | 0a8ecf4 |
| 22.8 | 1 | 0 | défaut réel | parseur hexadécimal (errno supprimé) | 38e5c64 |
| 22.10 | 0 | 0 | défaut réel (introduite puis corrigée) | parseur hexadécimal | 38e5c64 |
| 8.9 | 2 | 2 | violation de norme | à faire | — |
| 8.7 | 2 | 2 | faux positif | suppression ciblée | — |
| 21.6 | 2 | 2 | dérogation | DEV-002 (existant) | — |
| 7.4 | 1 | 1 | faux positif | cppcheck-suppress | — |
| 2.5 | 1 | 1 | à examiner | voir node.c (set_period) | — |
| 21.10 | 1 | 1 | dérogation | DEV-006 (à créer) | — |
| 21.1 | 1 | 1 | dérogation | DEV-006 (à créer) | — |
