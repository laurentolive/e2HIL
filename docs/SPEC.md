# e2HIL — Spécification

Version 0.1 (2026-10-21) — périmètre initial, à préciser avant la v1. Le travail restant est suivi dans les
[issues](https://github.com/laurentolive/e2HIL/issues).

## 1. Objectifs et périmètre

### 1.1 Objectifs
e2HIL exécute les **tests d'intégration et les tests système sur les vraies cartes**, à la main ou en intégration
continue. Il complète la gamme eSquareSoft :

| Outil | Rôle |
|---|---|
| Polenta | exigences, cas de test, campagnes, traçabilité |
| e2Unit | tests unitaires de fonctions C, sur PC et sur cible |
| e2Sim | intégration en simulation PC (firmware inchangé, HAL simulée, temps virtuel) |
| **e2HIL** | **intégration sur banc : cartes réelles, stimuli et mesures externes** |
| eScope | observation des variables du firmware en temps réel |

Principes :

1. **Zéro impact sur le firmware.** Le firmware testé est le firmware livré. e2HIL agit et mesure **de
   l'extérieur** (boîtier d'E/S, sonde, ports série). Lire des variables par la sonde est permis, pas en ajouter.
2. **Même forme de scénario qu'e2Sim.** Un test écrit pour e2Sim (`dut["BTN_POWER"].press(ms=300)`,
   `dut.run(s=1)`, `dut["PWM_MOTOR"].duty`) se réécrit à l'identique pour le banc, avec les **mêmes noms de
   signaux** que le `board.yaml` d'e2Sim. C'est une **convention** documentée, pas une dépendance de code :
   e2HIL est indépendant d'e2Sim, d'e2Unit et d'eScope (même règle que celle d'e2Sim vis-à-vis d'e2Unit).
3. **La vérité terrain vient du banc.** Fréquences, rapports cycliques et latences sont mesurés par le boîtier
   d'E/S, sans passer par le firmware testé.
4. **Fait pour la CI.** Un banc est une ressource partagée : réservation, remise sous tension à froid, journaux
   de tous les ports série attachés au rapport, verdict par code retour, JUnit.
5. **Simple et bon marché.** Un PC, des sondes du commerce, un boîtier d'E/S à base de Raspberry Pi Pico, un hub
   USB à coupure par port. Pas de châssis propriétaire.

### 1.2 Hors périmètre
- Simulation physique des capteurs (modèles thermiques, électriques) : c'est le rôle d'e2Sim.
- Tests de puissance, CEM, RF ; sécurité électrique du banc (laissée au montage).
- Vision par caméra des afficheurs (éventuellement en v2 ; en v1 l'image vient de la mémoire par la sonde).
- Gestion d'une ferme de bancs à grande échelle (v1 : un PC, un ou quelques bancs).

## 2. Glossaire
- **Banc** : un PC hôte, ses sondes, son boîtier d'E/S, son alimentation pilotable et les cartes branchées.
- **DUT** (*device under test*) : une carte du produit, avec son firmware.
- **Signal** : un fil du produit vu par le banc, désigné par son **nom produit** (`BTN_POWER`, `PWM_MOTOR`).
- **Voie** : une broche du boîtier d'E/S.
- **Boîtier d'E/S** (*I/O box*) : RP2040 avec le firmware `iobox/` d'e2HIL, piloté en USB CDC.

## 3. Architecture

```
                         PC hôte (Windows d'abord, Linux souhaité)
 ┌────────────────────────────────────────────────────────────────────────────┐
 │ pytest + plugin e2hil ── API scénario (dut["X"]...) ── rapports JUnit/Excel│
 │        │                                                                    │
 │  gestionnaire de banc : verrou, alimentation, flash, capture des ports série│
 └──┬──────────────┬────────────────┬──────────────────┬──────────────────────┘
    │ USB          │ USB            │ USB              │ USB (CDC)
 J-Link (SWD)   Arduino (bootloader  hub à coupure     boîtier d'E/S (Pico)
    │           série / UPDI)        par port           │ boutons, DAC, injection de fautes,
    ▼              ▼                  │ alimentation    │ mesure fréquence / rapport cyclique / latence
 carte A        carte B   ◄───────────┘                 ▼
    ▲──────────────▲──────────────────────────────── signaux du produit
```

## 4. Description du banc : `bench.yaml`

Une seule source pour tous les outils lancés sur le banc (e2Unit, eScope, scripts). Les ressources USB sont
désignées par **numéro de série**, jamais par l'ordre d'énumération ou le numéro de port COM.

```yaml
bench: fan-bench-1
iobox: {serial: E6614103E7}                # Pico, port CDC trouvé par numéro de série
power:
  hub: {type: uhubctl, location: "1-1"}    # ou {type: relay, port: ...}
duts:
  hmi:
    board: fpb-ra4e1
    flash: {jlink: {serial: 831000123, device: R7FA4E10D}}
    power: {hub_port: 2}
    uarts: {UART_DEBUG: {usb_serial: A10KX9Q2, baud: 115200}}
    symbols: ../build/ra4e1/fan_hmi.elf     # lecture de variables et du framebuffer par la sonde
  motor:
    board: arduino-nano-every
    flash: {arduino: {fqbn: "arduino:megaavr:nona4809", usb_serial: 7D3F...}}
    power: {hub_port: 3}
    uarts: {UART_SCOPE: {usb_serial: 7D3F..., baud: 115200}}
signals:                                    # mêmes noms que le board.yaml d'e2Sim
  BTN_POWER:  {iobox: GP2, kind: button, active: low}        # sortie open-drain en parallèle du bouton
  VBAT_ADC:   {iobox: I2C0, kind: dac, device: mcp4725, scale: "mV / 4.0"}
  FAULT_IN:   {iobox: GP6, kind: inject}                      # forçage d'une ligne
  PWM_MOTOR:  {iobox: GP10, kind: pwm_in}                     # mesure fréquence + rapport cyclique
  TACHO:      {iobox: GP11, kind: freq_in}
  SELF_HOLD:  {iobox: GP12, kind: bit_in}
```

## 5. Gestion du banc
- **Réservation** : verrou par banc (fichier verrou + PID), file d'attente, libération garantie en fin de job.
- **Alimentation** : marche / arrêt / cycle par DUT ; démarrage à froid avant chaque campagne.
- **Flash** : J-Link (SEGGER J-Link Commander ou pylink, sonde choisie par numéro de série) ; Arduino
  (arduino-cli / avrdude) ; méthode extensible (pyOCD, ST-LINK) par greffon.
- **Ports série** : capturés en continu pendant toute la campagne, horodatés, joints au rapport de chaque test ;
  un test peut attendre un motif (`dut.uart("UART_DEBUG").expect("READY", s=2)`).
- **Lecture mémoire par la sonde** (J-Link) : `dut.var("app.speedLevel")` avec les symboles ELF ; framebuffer lu
  et comparé à une image de référence PNG (la même que dans e2Sim quand le format de pixel est le même).

## 6. API de scénario
Convention commune avec e2Sim (même vocabulaire, mêmes noms de signaux), implémentée indépendamment :

| Appel | Banc |
|---|---|
| `dut["BTN_POWER"].press(ms=300)` | voie open-drain tirée pendant 300 ms |
| `dut["VBAT_ADC"].set(9200)` | DAC réglé pour 9,2 V après le pont diviseur déclaré |
| `dut["FAULT_IN"].inject(level=0, us=500)` | forçage d'une ligne |
| `dut["PWM_MOTOR"].duty`, `.freq` | mesurés par le boîtier |
| `dut["SELF_HOLD"] == 1` | lecture d'une entrée |
| `dut.run(s=1)` | attente **réelle** (pas de temps accéléré sur banc) |
| `dut.latency("FAULT_IN", "PWM_MOTOR", edge="fall")` | latence mesurée par le boîtier (résolution 1 µs visée) |
| `dut.var("app.speedLevel")` | lecture RAM par la sonde |
| `dut.power_cycle()`, `dut.reset()` | alimentation, reset par la sonde ou la broche |

Ce qui n'a de sens qu'en simulation (temps accéléré, forçage de la NVM, modèles Python) est marqué
`@e2hil.sim_only` dans un fichier de scénario partagé et ignoré par e2HIL.

## 7. Boîtier d'E/S (`iobox/`, MIT)
- RP2040 (Raspberry Pi Pico), firmware C (pico-sdk), protocole texte ou binaire simple sur USB CDC, versionné.
- Sorties open-drain (boutons), sorties logiques, DAC I2C (MCP4725), entrées logiques.
- Mesures par PIO : fréquence, rapport cyclique, comptage, horodatage de fronts, latence entre deux voies.
- Niveaux 3,3 V ; adaptation 5 V (Arduino) par résistances ou translateur documenté.
- Identification : version du protocole et du firmware vérifiée par l'hôte à la connexion.

## 8. pytest et rapports
- Plugin pytest : fixture `dut`, options `--e2hil-bench bench.yaml`, `--e2hil-flash` (flasher avant).
- Exigences : `@e2hil.requirement("SYS-0042")` ; rapport Excel exigences → tests (mise en page commune
  aux outils) ; JUnit avec propriétés (exigences, banc, versions des firmwares) importable dans Polenta.
- Code retour : 0 = PASS, 1 = FAIL, 2 = erreur de banc (sonde absente, verrou, flash) — une erreur de banc
  n'est jamais un échec du produit.

## 9. Intégration continue
- Le banc est un **runner GitHub Actions auto-hébergé** (label `hil`).
- **Sécurité** : sur un dépôt public, aucun job de banc sur une PR venant d'un fork (exécution de code
  arbitraire sur la machine du banc). Jobs de banc limités à `push` sur la branche principale, tags,
  déclenchement manuel et planifié, dans un *environment* GitHub protégé.
- **Session interactive** : le runner tourne dans une session ouverte (ouverture de session automatique), pas
  comme service, pour qu'eScope (WinForms) puisse s'y lancer.
- e2Unit (runner `jlink`) et eScope (automation REST) utilisent les sondes et ports décrits par `bench.yaml`
  pendant que le verrou est tenu.

## 10. Projet de référence
[fan-demo](https://github.com/laurentolive/fan-demo) : ventilateur à deux cartes (IHM sur Renesas
FPB-RA4E1 avec J-Link, carte moteur sur Arduino Nano Every en série). Ses tests système e2Sim sont repris sur
le banc avec les mêmes noms de signaux ; chaque fonctionnalité d'e2HIL y a une ligne dans la matrice de
couverture (`docs/MATRIX.md`).

## 11. Licence
Même modèle que les autres outils : outil PC sous PolyForm Noncommercial avec licence professionnelle pour
l'usage commercial ; firmware du boîtier d'E/S et exemples sous MIT.

## 12. Phasage

| Phase | Contenu |
|---|---|
| v0 | `bench.yaml`, verrou, alimentation, flash J-Link et Arduino, capture des ports série, CLI `e2hil` |
| v1 | boîtier d'E/S (boutons, DAC, entrées, mesures PIO), API de scénario, plugin pytest, JUnit/Excel, runner CI |
| v2 | latences fines, lecture mémoire et framebuffer, Linux, plusieurs bancs, caméra éventuelle |
