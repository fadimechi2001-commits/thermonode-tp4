# Dérogations MISRA C:2012 — ThermoNode

| ID | Règle | Fichier | Justification | Approbation |
|---|---|---|---|---|
| DEV-001 | 11.4 (conversion pointeur ↔ entier) | hal/hal_stm32.c (via STM32Cube) | Accès aux registres mappés en mémoire par la bibliothèque HAL du fabricant | Responsable qualité |
| DEV-002 | 21.6 (`<stdio.h>`) | src/frame.c, src/node.c | `snprintf` utilisé pour la sérialisation JSON avec contrôle de la longueur ; alternative (sérialisation manuelle) jugée plus risquée | Responsable qualité |
| DEV-003 | 21.6 (`<stdio.h>`) | hal/hal_sim.c | Code de simulation hôte, non embarqué | — |
| DEV-004 | 15.5 (un seul point de sortie) | src/* | retours anticipés des fonctions de garde (paramètres invalides, tampon trop petit) : lisibilité et contrôle immédiat | en attente d'approbation (enseignante) |
| DEV-005 | 18.4 (pas d'arithmétique de pointeur) | src/frame.c | parcours et découpe de la trame JSON, bornés par `len` et contrôlés avant chaque accès | en attente d'approbation (enseignante) |
