# e2HIL

[English](README.md) | **Français**

Tests de firmware embarqué **sur les vraies cartes** (*hardware in the loop*), à la main ou en intégration continue.

> État : spécification (v0.1). Rien à exécuter pour l'instant. Voir la [spécification](docs/SPEC.md) et les
> [issues](https://github.com/laurentolive/e2HIL/issues).

- **Zéro impact sur le firmware** : le banc agit et mesure de l'extérieur (boîtier d'E/S, sonde, ports série).
- **Même forme de scénario qu'e2Sim** : `dut["BTN_POWER"].press(ms=300)`, `dut.run(s=1)`,
  `dut["PWM_MOTOR"].duty`, avec les noms de signaux du `board.yaml` d'e2Sim. C'est une convention commune, pas
  une dépendance de code.
- **La vérité terrain vient du banc** : fréquence, rapport cyclique et latence sont mesurés par un boîtier d'E/S
  RP2040, pas par le firmware testé.
- **Fait pour la CI** : `bench.yaml` (sondes et ports désignés par numéro de série), verrou de banc, remise sous
  tension à froid, flash J-Link et Arduino, journaux série joints à chaque test, plugin pytest, rapports JUnit
  et Excel par exigence.
- **Bon marché** : un PC, des sondes du commerce, un Raspberry Pi Pico, un hub USB à coupure par port.

Il complète les outils eSquareSoft : Polenta (exigences), e2Unit (tests unitaires), e2Sim (simulation),
eScope (observation). Projet de référence : [fan-demo](https://github.com/laurentolive/fan-demo).

## Arborescence (prévue)

```
e2hil/      package Python : gestion du banc, flash, API de scénario, plugin pytest, rapports, CLI
iobox/      firmware du boîtier d'E/S RP2040 (MIT)
docs/       spécification, tutoriels
```

## Licence

Outil PC : PolyForm Noncommercial 1.0.0 ([LICENSE.md](LICENSE.md)), licence professionnelle pour l'usage
commercial. Firmware du boîtier d'E/S : MIT ([iobox/LICENSE.md](iobox/LICENSE.md)).
