# HANDOFF — CarSystem (reprise sur Asahi Linux)

Document de passation pour un agent qui reprend le travail sans avoir vu la conversation précédente.
Rédigé le 2026-10-26 depuis macOS. L'utilisateur parle **français** : réponds en français.

## 1. Qui, quoi, pourquoi

- Utilisateur : étudiant (BUT), projet **CarSystem** (logiciel automobile / bus CAN). Dossier de travail : `Projets/CarSystem`.
- Le dossier `Apprends/` (non suivi par git, `?? Apprends/`) contient des dépôts clonés pour **apprendre et réutiliser** (voir §3).
- Objectif de fond : construire un **banc d'essai CAN / ECU** sur Linux (SocketCAN), en réutilisant de l'open source plutôt que de tout réécrire.
- L'utilisateur passe sur **Asahi Linux** (Fedora Asahi Remix, Linux aarch64 sur Apple Silicon). Il avait d'abord pensé à une VM Colima sous macOS ; Asahi la rend inutile.

## 2. Décisions déjà prises / conclusions

1. Pas de clone de CANoe / DaVinci / EB tresos / VectorCAST / Simulink : trop gros, et leur valeur est la certification. On fait une version **lite** d'un seul outil : l'équivalent de CANoe (bus + simulateur d'ECU + scripts + DBC).
2. **EcuBus-Pro fait déjà une grande partie de ça** → on le prend comme **base de travail** et on ajoute ce qui manque, plutôt que de réécrire `vcan_macOS` en Rust.
3. Sous Linux, **SocketCAN (`vcan`, `can-utils`) est natif** → plus besoin de `vcan_macOS`, de `macos-vcan` ni de `libPCBUSB`.
4. **Ce qui manque et qu'il faut coder** : un **pilote SocketCAN pour EcuBus-Pro** (voir §5). Le reste de la compilation devrait être de la configuration.
5. Priorité ensuite (idées, pas encore décidées) : simulateur d'ECU avec watchdog / mode dégradé / injection de fautes ; messages cycliques avec compteur + CRC + timeout (profils E2E) ; traçabilité HARA → exigences → tests ; MAC sur trames (SecOC-like). Hors périmètre : CAN-XL, certification ISO 26262 (aucun outil ici n'est qualifié), vrai temps réel dur.

## 3. Les dépôts dans `Apprends/`

| Dépôt | Licence | Verdict |
|---|---|---|
| `EcuBus-Pro` | **Apache-2.0** (`license.txt`, `package.json` v0.8.67). Pas de NOTICE. | **Base du projet.** Electron + TypeScript + Vue, ~443 fichiers TS/Vue. CAN/CAN-FD, LIN, DoIP, SOME/IP, UDS, scripts TS, tests HIL, DBC (lecture), LDF (édition), CLI. |
| `vcan_macOS` | **Aucune licence trouvée** (= droits réservés) | Démon userspace C (UNIX datagram) : `macos-vcand`, `libmacos_vcan`, `vcan-send`, `vcan-dump`. Bon à lire. **Ne pas copier de code** sans licence/accord de l'auteur. Inutile sous Linux. |
| `macos-vcan` | Aucune licence trouvée | Extension noyau (kext) macOS. **Impasse** : pas de famille `PF_CAN` côté macOS, kext déprécié, doc incohérente (`CLAUDE.md` décrit un `linux_vcan.c` absent). Ignorer. |
| `PCBUSB-Utilities` | GPL-2.0 + licence freeware pour la lib | `can_moni` / `can_test` pour adaptateurs **PEAK** via `libPCBUSB` (propriétaire, non redistribuable). Inutile sous Linux (le pilote `peak_usb` est dans le noyau). GPL-2.0 : ne pas mélanger avec du GPL-3 si "2.0 seulement". |
| `ZedSuite` | **GPL-3.0** + marque protégée (`TRADEMARKS.md`) | Éditeur de cartographies moteur Bosch EDC15/16 (tuning, "DTC off", correction de checksum). **Hors sujet** pour le CAN, légalité douteuse sur route. Lecture seule. L'auto-updater exécute un binaire GitHub sans vérification de signature ; **ne jamais lancer `install-macos.sh`** (`curl \| sh` + retrait de la quarantaine). |

Règles de licence : Apache-2.0 → GPL-3.0 possible, l'inverse non. Copier du code Apache = garder licence et mentions, signaler les modifs. Pour ses propres ajouts, l'utilisateur gagne à rester en **Apache-2.0**.

## 4. EcuBus-Pro : ce qui a été constaté

- Pilotes déclarés par OS dans `package.json` (`ecubusPro.vendor`) : win32 = simulate, kvaser, peak, zlg, toomoss, vector, slcan, ecubus, candle ; **linux et darwin = `simulate`, `slcan` seulement**. Il n'y a **pas de SocketCAN**.
- Les pilotes vivent dans `src/main/docan/<nom>/` (`simulate/index.ts`, `slcan/index.ts`, …). Le pilote SLCAN utilise la lib npm `serialport` (`SerialPort({ path })`), pas de socket réseau.
- `resources/lib/` contient des **DLL Windows de constructeurs** (`PCANBasic`, `vxlapi64`, `canlib32`, `zlgcan`, `USB2XXX`, `vsomeip3*`) : inutiles hors Windows, licences distinctes de l'Apache-2.0. Dans un fork destiné à Linux, les supprimer évite tout souci de redistribution (non vérifié juridiquement).
- Sous Linux, `src/main/docan/binding.gyp` ne compile qu'un fichier factice `fake_linux.cxx` (15 lignes) à la place des pilotes constructeurs. `npm run native` = `docan` + `dolin` + `someip` (`node-gyp rebuild`). **`vsomeip` est le module le plus risqué sur ARM** (vraie lib C++) ; non nécessaire pour le CAN, on peut l'ignorer.
- CI officielle sur `ubuntu-latest` (x86_64) : **pas de test ARM connu**. Node 22 utilisé côté macOS (v22.23.1).
- `CLAUDE.md` et `AGENTS.md` du dépôt décrivent les commandes (`npm run dev`, `npm run test`, `npm run worker:js` obligatoire après un changement dans `src/main/worker/`, etc.). Les lire avant de modifier.

## 5. Plan sur Asahi (dans l'ordre)

1. **Vérifier SocketCAN** : `sudo modprobe vcan && sudo ip link add dev vcan0 type vcan && sudo ip link set up vcan0`, puis `sudo dnf install can-utils`, `candump vcan0` dans un terminal et `cansend vcan0 123#DEADBEEF` dans un autre. Si `modprobe` échoue, le noyau Asahi n'a pas les modules : le dire à l'utilisateur.
2. **Compiler EcuBus-Pro** : `cd Apprends/EcuBus-Pro && npm install && npm run dev` (Node 22). Si une étape native échoue, ne pas lancer `npm run native` en entier ; ignorer `vsomeip`. Noter chaque erreur et la correction (souvent une variable d'env ou une version, pas du code).
3. **Écrire le pilote `socketcan`** dans `src/main/docan/socketcan/`, sur le modèle de `simulate/` et `slcan/` : ajouter `"socketcan"` à la liste Linux de `package.json`, déclarer le vendor là où les autres sont enregistrés (chercher `slcan` dans `src/` pour trouver tous les points d'enregistrement). Node n'a pas de support natif de `PF_CAN` : deux options, à choisir avec l'utilisateur — module natif N-API minimal, ou petit processus compagnon (Rust ou C) parlant en JSON/stdio. Recommandation : **processus compagnon**, plus simple à compiler sur ARM.
4. Test bout en bout : trame envoyée depuis EcuBus-Pro vers `vcan0`, vue avec `candump`, et inversement ; puis CAN-FD.
5. Ensuite seulement : simulateur d'ECU, E2E, traçabilité, sécurité.

## 6. Ce qui n'a PAS été vérifié

- Rien n'a été compilé ni exécuté sur macOS : le **sandbox** de l'agent précédent bloquait le cache `xcrun` et `/tmp` (donc `make` de `vcan_macOS` non validé).
- Aucune affirmation sur Asahi n'a été testée : présence de `can`/`vcan` dans le noyau, `PREEMPT_RT`, compilation arm64 d'EcuBus-Pro, état de FEX/`muvm`.
- Licence de `vsomeip` (probablement MPL-2.0) à confirmer.
- Origine et auteur de `vcan_macOS` inconnus : l'utilisateur ne l'a pas précisé. Demande-lui s'il veut que ce code soit réutilisé.

## 7. Préférences et contexte de travail

- Réponses en **français**, concises, avec un avis clair plutôt qu'un catalogue d'options.
- L'utilisateur utilise un plugin "ponytail" (mode « solution la plus simple / pas de sur-ingénierie ») ; l'esprit : ne pas ajouter d'abstraction ni de dépendance inutile, une vérification exécutable pour toute logique non triviale.
- Ne pas commiter ni pousser sans demande explicite. `Apprends/` n'est pas suivi par git ; ce fichier non plus.
- Aucun logiciel de ce banc n'est qualifié ISO 26262 : ne jamais présenter le résultat comme apte à valider du code de véhicule réel. Pour ZedSuite, ne rien exécuter sur un vrai véhicule.
