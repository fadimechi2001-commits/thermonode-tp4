# Revue de MR !7 « fix hysteresis » — réviseurs : <A>, <B>

| # | Fichier:ligne | Constat | Règle / source | Gravité | Demande à l'auteur |
|---|---|---|---|---|---|
| 1 | firmware/src/threshold.c:31 | Constante `500` codée en dur à la place de `hyst_centi` : l'hystérésis configurée est ignorée. `threshold_set(6000, 300)` puis `threshold_step(5699, true)` renvoie NONE au lieu de CLEAR. | EF-04, liste « constantes » | bloquant | Utiliser `hyst_centi`. Voir DR-TP4-01. |
| 2 | firmware/test/test_threshold.c (renommage `xtest_…`, `RUN_TEST` commenté « flaky ») | Le test qui couvre ce cas est désactivé. C'est pour cela que la CI est verte. | liste « tests » | bloquant | Réactiver `test_EF04_configured_hysteresis_is_used`. Un test instable se corrige, il ne se désactive pas. |
| 3 | firmware/src/node.c:22 | `strncpy(node_id, id, TN_ID_LEN)` supprime le terminateur. Un identifiant de 8 caractères ou plus laisse `node_id` sans `'\0'`. | liste « chaînes et tampons » | bloquant | Restaurer `TN_ID_LEN - 1u` et `node_id[TN_ID_LEN - 1u] = '\0'`. Voir DR-TP4-02. |
| 4 | firmware/src/threshold.c:34, 36, 38 | Accolades supprimées (MISRA 15.6). `ev = THR_CLEAR, above = 0u;` utilise l'opérateur virgule (MISRA 12.3). L'outil signale aussi 12.1 à la ligne 31. | MISRA C:2012 15.6, 12.3, 12.1 | mineure | Rétablir les accolades et séparer les instructions. |
| 5 | Message du commit « fix hysteresis » | Titre sans type, portée ni référence EF/DEF. Le diff touche `node.c` et réécrit la branche hors alerte sans que cela soit annoncé. Le refactor est équivalent (vérifié), mais il doit être dans un commit séparé. | liste « périmètre », « message de commit » | mineure | Découper : `fix(node)`, `fix(threshold)`, `refactor(threshold)`, avec référence EF-04. |

## Rapports de défaut

### DR-TP4-01 — Hystérésis codée en dur
- Commit : `78f2013` (branche `fix/EF-04-hysteresis`, base `019db8e`)
- Environnement : VM TP0, gcc 13.3, cppcheck 2.13
- Étapes : `threshold_set(6000, 300)` puis `threshold_step(5699, true)`
- Attendu : `THR_CLEAR`
- Obtenu : `THR_NONE`
- Sévérité : élevée (paramètre configuré ignoré, alarme levée trop tard)
- Priorité : haute, bloque la fusion
- Références : EF-04 ; liste « constantes »
- Pièce jointe : sortie de `run_validation.sh` (`test_V1_configured_hysteresis_used` : Expected 2 Was 0)

### DR-TP4-02 — Terminateur de `node_id` perdu
- Commit : `78f2013`
- Étapes : `node_set_id("ABCDEFGHIJ")`, puis inspection de `node_id[0..7]`
- Attendu : un `'\0'` dans les 8 octets
- Obtenu : aucun terminateur ; les lectures en chaîne dépassent le tampon
- Sévérité : élevée (lecture hors bornes)
- Priorité : haute
- Références : liste « chaînes et tampons »
- Pièce jointe : sortie de `run_validation.sh` (`node_id ... NO`)

## Validation effectuée
- `make -C firmware test` : 2 tests, 0 échec (suite `test_convert` seule dans ce squelette)
- `run_validation.sh ~/tn` : `main` 6/6 OK, terminateur présent ; `fix/EF-04-hysteresis` V1 en échec, terminateur absent
- cppcheck + addon MISRA : 15.6 (lignes 34, 36, 38) et 12.1 (ligne 31) relevés. Les `unusedFunction` sont du bruit lié au périmètre de compilation.

## Décision

**Demander des changements.** Synthèse pour l'auteur : la hystérésis configurée est court-circuitée par une constante (DR-TP4-01), le test qui le couvre est désactivé et doit être réactivé, et le terminateur de `node_id` est perdu (DR-TP4-02). Corriger aussi les accolades MISRA et l'opérateur virgule, et séparer les changements en commits au titre conventionnel.
